from typing import Final

TOKEN: Final = '8058587021:AAE0wrQLeqc2DbqP9r3Jp8VJgnVfkYHe5KA'
BOT_USERNAME: Final = '@Traaading_bot'
import os
import time
import logging
import requests
import numpy as np
from solders.keypair import Keypair
from solders.pubkey import Pubkey
from solana.rpc.api import Client
from solana.transaction import Transaction
from spl.token.instructions import TransferParams, transfer
from telegram import Bot
from decimal import Decimal

# Configuration
DEX_SCREENER_SOLANA_API = "https://api.dexscreener.com/latest/dex/chains/solana"
JUPITER_QUOTE_API = "https://quote-api.jup.ag/v6/quote"
JUPITER_SWAP_API = "https://quote-api.jup.ag/v6/swap"
SAFETY_CHECK_API = "https://api.solanasniffer.xyz/token/{}"
TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN")
SOLANA_RPC_URL = os.getenv("SOLANA_RPC_URL")
WALLET_PRIVATE_KEY = os.getenv("PRIVATE_KEY")

# Trading Parameters
MIN_MCAP = 100000  # $100K
MAX_MCAP = 500000  # $500K
MIN_LIQUIDITY = 100000  # $100K
TAKE_PROFIT = 1.20  # 20%
STOP_LOSS = 0.85  # 15%
TRADE_SLIPPAGE = 2  # 2%
MAX_POSITION_SIZE = 0.1  # 10% of portfolio
VOLATILITY_WINDOW = 5  # 5-minute window for volatility calculation
TRAILING_TAKE_PROFIT = 0.05  # 5% trailing take-profit
WHALE_TRANSACTION_THRESHOLD = 10000  # $10K

class SolanaTrader:
    def __init__(self):
        self.client = Client(SOLANA_RPC_URL)
        self.keypair = Keypair.from_base58_string(WALLET_PRIVATE_KEY)
        self.wallet = self.keypair.pubkey()
        self.bot = Bot(token=TELEGRAM_TOKEN)
        self.open_positions = {}
        self.usdc_balance = self.get_usdc_balance()
        self.price_history = {}  # Store price history for volatility calculation

    def get_usdc_balance(self):
        """Get USDC balance from wallet"""
        usdc_mint = Pubkey.from_string("EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v")
        return self.get_token_balance(usdc_mint)

    def get_token_balance(self, mint: Pubkey):
        """Get token balance from wallet"""
        resp = self.client.get_token_accounts_by_owner_json_parsed(
            self.wallet,
            mint=mint
        ).value
        return sum(int(acc.account.data.parsed["info"]["tokenAmount"]["amount"]) 
                 for acc in resp) / 10**6

    def execute_swap(self, input_mint: str, output_mint: str, amount: float):
        """Execute swap using Jupiter API"""
        try:
            # Get best quote
            quote_params = {
                "inputMint": input_mint,
                "outputMint": output_mint,
                "amount": int(amount * 10**6),
                "slippageBps": TRADE_SLIPPAGE * 100
            }
            quote = requests.get(JUPITER_QUOTE_API, params=quote_params).json()
            
            # Execute swap
            swap_data = {
                "quoteResponse": quote,
                "userPublicKey": str(self.wallet),
                "wrapUnwrapSOL": True
            }
            swap_instruction = requests.post(JUPITER_SWAP_API, json=swap_data).json()
            
            # Build and send transaction
            txn = Transaction().add(swap_instruction)
            result = self.client.send_transaction(txn, self.keypair)
            return result.value
        except Exception as e:
            logging.error(f"Swap failed: {e}")
            return None

    def buy_token(self, token_address: str, liquidity: float):
        """Execute buy order for target token"""
        try:
            # Calculate position size based on liquidity
            position_size = min(self.usdc_balance * MAX_POSITION_SIZE * (liquidity / MIN_LIQUIDITY), 
                              self.usdc_balance * 0.99)  # Leave room for fees
            
            # Execute USDC to Token swap
            result = self.execute_swap(
                input_mint="EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
                output_mint=token_address,
                amount=position_size
            )
            
            if result:
                self.track_position(token_address, position_size)
                self.send_alert(f"✅ Bought {token_address} - TX: {result}")
                return True
            return False
        except Exception as e:
            logging.error(f"Buy failed: {e}")
            return False

    def sell_token(self, token_address: str):
        """Execute sell order for target token"""
        try:
            token_balance = self.get_token_balance(Pubkey.from_string(token_address))
            if token_balance > 0:
                result = self.execute_swap(
                    input_mint=token_address,
                    output_mint="EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
                    amount=token_balance
                )
                if result:
                    self.send_alert(f"💰 Sold {token_address} - TX: {result}")
                    return True
            return False
        except Exception as e:
            logging.error(f"Sell failed: {e}")
            return False

    def track_position(self, token_address: str, entry_value: float):
        """Track position and set profit targets"""
        entry_price = self.get_token_price(token_address)
        self.open_positions[token_address] = {
            'entry_price': entry_price,
            'entry_time': time.time(),
            'position_size': entry_value,
            'take_profit': entry_price * TAKE_PROFIT,
            'stop_loss': entry_price * STOP_LOSS,
            'trailing_take_profit': entry_price * (1 + TRAILING_TAKE_PROFIT),
            'price_history': [entry_price]  # Initialize price history for volatility calculation
        }

    def monitor_positions(self):
        """Monitor open positions for profit targets"""
        for token_address, position in list(self.open_positions.items()):
            current_price = self.get_token_price(token_address)
            
            # Update price history for volatility calculation
            position['price_history'].append(current_price)
            if len(position['price_history']) > VOLATILITY_WINDOW:
                position['price_history'].pop(0)
            
            # Calculate volatility
            volatility = np.std(position['price_history']) / np.mean(position['price_history'])
            
            # Adjust stop-loss based on volatility
            dynamic_stop_loss = position['entry_price'] * (1 - STOP_LOSS * (1 + volatility))
            position['stop_loss'] = max(position['stop_loss'], dynamic_stop_loss)
            
            # Update trailing take-profit
            if current_price > position['trailing_take_profit']:
                position['trailing_take_profit'] = current_price * (1 - TRAILING_TAKE_PROFIT)
            
            # Check for take-profit or stop-loss
            if current_price >= position['trailing_take_profit']:
                self.sell_token(token_address)
                del self.open_positions[token_address]
            elif current_price <= position['stop_loss']:
                self.sell_token(token_address)
                del self.open_positions[token_address]

    def get_token_price(self, token_address: str) -> float:
        """Get current token price from DEX Screener"""
        try:
            response = requests.get(f"{DEX_SCREENER_SOLANA_API}/tokens/{token_address}")
            return float(response.json()['priceUsd'])
        except Exception as e:
            logging.error(f"Price fetch failed: {e}")
            return 0.0

    def send_alert(self, message: str):
        """Send alert via Telegram"""
        self.bot.send_message(chat_id=os.getenv("TELEGRAM_CHAT_ID"), text=message)

class TokenScanner:
    def __init__(self):
        self.trader = SolanaTrader()
        self.last_scan = 0

    def scan_new_pairs(self):
        """Scan for new Solana pairs on DEX Screener"""
        try:
            response = requests.get(DEX_SCREENER_SOLANA_API)
            pairs = response.json().get('pairs', [])
            
            for pair in sorted(pairs, key=lambda x: x['pairCreatedAt'], reverse=True):
                if self.is_valid_pair(pair):
                    if self.safety_check(pair['baseToken']['address']):
                        self.trader.buy_token(pair['baseToken']['address'], pair['liquidity']['usd'])
        except Exception as e:
            logging.error(f"Scan error: {e}")

    def is_valid_pair(self, pair: dict) -> bool:
        """Validate trading pair against criteria"""
        pair_age = (time.time() - pair['pairCreatedAt']/1000) / 60  # Age in minutes
        return all([
            pair['chainId'] == 'solana',
            pair['fdv'] > MIN_MCAP,
            pair['fdv'] < MAX_MCAP,
            pair['liquidity']['usd'] > MIN_LIQUIDITY,
            pair['volume']['h24'] > MIN_LIQUIDITY * 0.5,
            pair_age < 60  # Less than 1hr old
        ])

    def safety_check(self, contract_address: str) -> bool:
        """Perform comprehensive safety check"""
        try:
            response = requests.get(SAFETY_CHECK_API.format(contract_address))
            data = response.json()
            return all([
                data['is_mintable'] is False,
                data['holder_count'] > 100,
                data['lp_locked'] > 90,
                data['honeypot'] is False
            ])
        except Exception as e:
            logging.error(f"Safety check failed: {e}")
            return False

    def track_whale_transactions(self):
        """Track large transactions (whale transactions)"""
        try:
            response = requests.get(f"{DEX_SCREENER_SOLANA_API}/whale-transactions")
            transactions = response.json().get('transactions', [])
            
            for tx in transactions:
                if tx['amountUsd'] > WHALE_TRANSACTION_THRESHOLD:
                    self.trader.send_alert(f"🐋 Whale transaction detected: {tx['amountUsd']} USD in {tx['token']}")
        except Exception as e:
            logging.error(f"Whale tracking failed: {e}")

def main():
    logging.basicConfig(level=logging.INFO)
    scanner = TokenScanner()
    
    while True:
        try:
            scanner.scan_new_pairs()
            scanner.trader.monitor_positions()
            scanner.track_whale_transactions()
            time.sleep(60)
        except KeyboardInterrupt:
            break
        except Exception as e:
            logging.error(f"Main loop error: {e}")
            time.sleep(30)

if __name__ == "__main__":
    main()
