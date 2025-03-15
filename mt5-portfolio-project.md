# MT5 Trading Data Analytics Portfolio Project

This portfolio project demonstrates the extraction of trading data from MetaTrader 5 (MT5), storing it in a database, and creating interactive visualizations with Microsoft Power BI. This end-to-end solution showcases skills in financial data processing, database management, and data visualization.

## Project Overview

### Architecture
1. **Data Extraction**: Python script connects to MT5 and extracts historical price data, indicators, and account information
2. **Data Storage**: SQL database (PostgreSQL) stores the extracted data in a structured format
3. **Data Visualization**: Power BI dashboard connects to the database and visualizes trading performance and market analysis

### Key Features
- Automated data extraction from MT5 on a scheduled basis
- Historical and real-time price data collection
- Technical indicator calculations
- Trade history and performance metrics
- Interactive Power BI dashboard with multiple visualization views

## Part 1: MT5 Data Extraction with Python

### Setup Environment

```python
# Install required packages
# pip install MetaTrader5 pandas sqlalchemy psycopg2-binary python-dotenv schedule

import MetaTrader5 as mt5
import pandas as pd
import numpy as np
import time
import datetime as dt
import schedule
import os
from dotenv import load_dotenv
from sqlalchemy import create_engine, text
```

### Create Environment Variables File (.env)

```
MT5_PATH=C:/Program Files/MetaTrader 5/terminal64.exe
MT5_LOGIN=your_login
MT5_PASSWORD=your_password
MT5_SERVER=your_broker_server
DB_USER=postgres
DB_PASSWORD=your_db_password
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mt5_trading_data
```

### MT5 Connection Functions

```python
def connect_to_mt5():
    """Connect to MT5 terminal"""
    load_dotenv()
    
    mt5_path = os.getenv("MT5_PATH")
    login = int(os.getenv("MT5_LOGIN"))
    password = os.getenv("MT5_PASSWORD")
    server = os.getenv("MT5_SERVER")
    
    # Initialize MT5 connection
    if not mt5.initialize(path=mt5_path):
        print(f"initialize() failed, error code = {mt5.last_error()}")
        return False
    
    # Login to MT5 account
    if not mt5.login(login, password, server):
        print(f"login failed, error code = {mt5.last_error()}")
        mt5.shutdown()
        return False
    
    print(f"Connected to MT5 account #{login}")
    return True

def disconnect_from_mt5():
    """Disconnect from MT5 terminal"""
    mt5.shutdown()
    print("Disconnected from MT5")
```

### Database Connection Functions

```python
def create_db_engine():
    """Create SQLAlchemy engine for database connection"""
    load_dotenv()
    
    db_user = os.getenv("DB_USER")
    db_password = os.getenv("DB_PASSWORD")
    db_host = os.getenv("DB_HOST")
    db_port = os.getenv("DB_PORT")
    db_name = os.getenv("DB_NAME")
    
    connection_string = f"postgresql://{db_user}:{db_password}@{db_host}:{db_port}/{db_name}"
    return create_engine(connection_string)

def create_database_tables(engine):
    """Create required database tables if they don't exist"""
    # Price data table
    engine.execute(text("""
    CREATE TABLE IF NOT EXISTS price_data (
        id SERIAL PRIMARY KEY,
        symbol VARCHAR(20) NOT NULL,
        timeframe VARCHAR(10) NOT NULL,
        timestamp TIMESTAMP NOT NULL,
        open FLOAT NOT NULL,
        high FLOAT NOT NULL,
        low FLOAT NOT NULL,
        close FLOAT NOT NULL,
        volume FLOAT NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(symbol, timeframe, timestamp)
    );
    """))
    
    # Technical indicators table
    engine.execute(text("""
    CREATE TABLE IF NOT EXISTS technical_indicators (
        id SERIAL PRIMARY KEY,
        symbol VARCHAR(20) NOT NULL,
        timeframe VARCHAR(10) NOT NULL,
        timestamp TIMESTAMP NOT NULL,
        rsi FLOAT,
        macd FLOAT,
        macd_signal FLOAT,
        macd_hist FLOAT,
        ma_20 FLOAT,
        ma_50 FLOAT,
        ma_200 FLOAT,
        bollinger_upper FLOAT,
        bollinger_middle FLOAT,
        bollinger_lower FLOAT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(symbol, timeframe, timestamp)
    );
    """))
    
    # Account data table
    engine.execute(text("""
    CREATE TABLE IF NOT EXISTS account_data (
        id SERIAL PRIMARY KEY,
        timestamp TIMESTAMP NOT NULL,
        balance FLOAT NOT NULL,
        equity FLOAT NOT NULL,
        margin FLOAT NOT NULL,
        free_margin FLOAT NOT NULL,
        margin_level FLOAT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    """))
    
    # Trades data table
    engine.execute(text("""
    CREATE TABLE IF NOT EXISTS trades_data (
        id SERIAL PRIMARY KEY,
        ticket INT NOT NULL,
        symbol VARCHAR(20) NOT NULL,
        order_type VARCHAR(10) NOT NULL,
        volume FLOAT NOT NULL,
        open_time TIMESTAMP NOT NULL,
        open_price FLOAT NOT NULL,
        sl FLOAT,
        tp FLOAT,
        close_time TIMESTAMP,
        close_price FLOAT,
        commission FLOAT,
        swap FLOAT,
        profit FLOAT,
        comment VARCHAR(255),
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(ticket)
    );
    """))
```

### Data Extraction Functions

```python
def extract_price_data(symbols, timeframes):
    """
    Extract historical price data for multiple symbols and timeframes
    
    Args:
        symbols (list): List of trading symbols (e.g., ["EURUSD", "GBPUSD"])
        timeframes (list): List of timeframes (e.g., [mt5.TIMEFRAME_M15, mt5.TIMEFRAME_H1])
    
    Returns:
        dict: Dictionary with key as (symbol, timeframe) and value as dataframe of price data
    """
    result = {}
    
    # Map timeframe constants to string names for database storage
    timeframe_names = {
        mt5.TIMEFRAME_M1: "M1",
        mt5.TIMEFRAME_M5: "M5",
        mt5.TIMEFRAME_M15: "M15",
        mt5.TIMEFRAME_M30: "M30",
        mt5.TIMEFRAME_H1: "H1",
        mt5.TIMEFRAME_H4: "H4",
        mt5.TIMEFRAME_D1: "D1",
        mt5.TIMEFRAME_W1: "W1",
        mt5.TIMEFRAME_MN1: "MN1"
    }
    
    for symbol in symbols:
        for timeframe in timeframes:
            tf_name = timeframe_names.get(timeframe, f"TF{timeframe}")
            
            # Get current time
            now = dt.datetime.now()
            
            # Get 1000 historical candles (adjust as needed)
            candles = mt5.copy_rates_from(symbol, timeframe, now, 1000)
            
            if candles is None or len(candles) == 0:
                print(f"Failed to get {symbol} {tf_name} data, error: {mt5.last_error()}")
                continue
            
            # Convert to pandas dataframe
            df = pd.DataFrame(candles)
            
            # Convert time (comes as a timestamp) to datetime
            df['time'] = pd.to_datetime(df['time'], unit='s')
            
            # Add symbol and timeframe info
            df['symbol'] = symbol
            df['timeframe'] = tf_name
            
            # Rename columns for database compatibility
            df = df.rename(columns={'time': 'timestamp'})
            
            result[(symbol, tf_name)] = df
            print(f"Extracted {len(df)} candles for {symbol} {tf_name}")
    
    return result

def calculate_technical_indicators(price_data):
    """
    Calculate technical indicators from price data
    
    Args:
        price_data (dict): Dictionary of price dataframes
    
    Returns:
        dict: Dictionary with key as (symbol, timeframe) and value as dataframe of indicators
    """
    result = {}
    
    for (symbol, timeframe), df in price_data.items():
        # Make a copy of the dataframe
        indicators_df = df[['timestamp', 'symbol', 'timeframe']].copy()
        
        # RSI (14)
        delta = df['close'].diff()
        gain = delta.where(delta > 0, 0)
        loss = -delta.where(delta < 0, 0)
        avg_gain = gain.rolling(window=14).mean()
        avg_loss = loss.rolling(window=14).mean()
        rs = avg_gain / avg_loss
        indicators_df['rsi'] = 100 - (100 / (1 + rs))
        
        # Moving Averages
        indicators_df['ma_20'] = df['close'].rolling(window=20).mean()
        indicators_df['ma_50'] = df['close'].rolling(window=50).mean()
        indicators_df['ma_200'] = df['close'].rolling(window=200).mean()
        
        # MACD
        ema_12 = df['close'].ewm(span=12, adjust=False).mean()
        ema_26 = df['close'].ewm(span=26, adjust=False).mean()
        indicators_df['macd'] = ema_12 - ema_26
        indicators_df['macd_signal'] = indicators_df['macd'].ewm(span=9, adjust=False).mean()
        indicators_df['macd_hist'] = indicators_df['macd'] - indicators_df['macd_signal']
        
        # Bollinger Bands
        indicators_df['bollinger_middle'] = df['close'].rolling(window=20).mean()
        std_dev = df['close'].rolling(window=20).std()
        indicators_df['bollinger_upper'] = indicators_df['bollinger_middle'] + (std_dev * 2)
        indicators_df['bollinger_lower'] = indicators_df['bollinger_middle'] - (std_dev * 2)
        
        result[(symbol, timeframe)] = indicators_df
        print(f"Calculated indicators for {symbol} {timeframe}")
    
    return result

def extract_account_info():
    """Extract account information"""
    account_info = mt5.account_info()
    if account_info is None:
        print(f"Failed to get account info, error: {mt5.last_error()}")
        return None
    
    # Convert to dictionary
    account_dict = {
        'timestamp': dt.datetime.now(),
        'balance': account_info.balance,
        'equity': account_info.equity,
        'margin': account_info.margin,
        'free_margin': account_info.margin_free,
        'margin_level': account_info.margin_level
    }
    
    return pd.DataFrame([account_dict])

def extract_trades_history():
    """Extract trades history (closed positions)"""
    # Get current time
    now = dt.datetime.now()
    
    # Get trades from the last 30 days
    from_date = now - dt.timedelta(days=30)
    
    # Convert to MT5 timestamp format
    from_timestamp = int(from_date.timestamp())
    to_timestamp = int(now.timestamp())
    
    # Get the history of closed orders
    history_orders = mt5.history_deals_get(from_timestamp, to_timestamp)
    
    if history_orders is None:
        print(f"No orders found, error: {mt5.last_error()}")
        return None
    
    # Convert to pandas dataframe
    orders_df = pd.DataFrame(list(history_orders), columns=history_orders[0]._asdict().keys())
    
    # Process and clean the data
    if not orders_df.empty:
        # Convert timestamps to datetime
        orders_df['time'] = pd.to_datetime(orders_df['time'], unit='s')
        
        # Rename columns for database compatibility
        orders_df = orders_df.rename(columns={
            'time': 'open_time',
            'price': 'open_price',
            'volume': 'volume',
            'symbol': 'symbol',
            'type': 'order_type',
            'profit': 'profit',
            'commission': 'commission',
            'swap': 'swap',
            'order': 'ticket',
            'comment': 'comment'
        })
        
        # Select relevant columns
        columns_to_keep = [
            'ticket', 'symbol', 'order_type', 'volume', 'open_time', 'open_price',
            'commission', 'swap', 'profit', 'comment'
        ]
        
        orders_df = orders_df[columns_to_keep]
        
        # Add empty columns for close_time and close_price
        orders_df['close_time'] = None
        orders_df['close_price'] = None
        orders_df['sl'] = None
        orders_df['tp'] = None
        
        print(f"Extracted {len(orders_df)} historical trades")
        return orders_df
    
    return None

def extract_open_positions():
    """Extract current open positions"""
    positions = mt5.positions_get()
    
    if positions is None:
        print(f"No positions found, error: {mt5.last_error()}")
        return None
    
    if len(positions) == 0:
        print("No open positions")
        return None
    
    # Convert to pandas dataframe
    positions_df = pd.DataFrame(list(positions), columns=positions[0]._asdict().keys())
    
    # Process and clean the data
    if not positions_df.empty:
        # Convert timestamps to datetime
        positions_df['time'] = pd.to_datetime(positions_df['time'], unit='s')
        
        # Rename columns for database compatibility
        positions_df = positions_df.rename(columns={
            'time': 'open_time',
            'price_open': 'open_price',
            'volume': 'volume',
            'symbol': 'symbol',
            'type': 'order_type',
            'ticket': 'ticket',
            'sl': 'sl',
            'tp': 'tp',
            'comment': 'comment'
        })
        
        # Select relevant columns
        columns_to_keep = [
            'ticket', 'symbol', 'order_type', 'volume', 'open_time', 'open_price',
            'sl', 'tp', 'comment'
        ]
        
        positions_df = positions_df[columns_to_keep]
        
        # Add empty columns for close data and financial results
        positions_df['close_time'] = None
        positions_df['close_price'] = None
        positions_df['commission'] = 0.0
        positions_df['swap'] = 0.0
        positions_df['profit'] = 0.0
        
        print(f"Extracted {len(positions_df)} open positions")
        return positions_df
    
    return None
```

### Database Storage Functions

```python
def store_price_data(engine, price_data_dict):
    """Store price data in database"""
    for (symbol, timeframe), df in price_data_dict.items():
        # Select only needed columns
        cols_to_store = ['timestamp