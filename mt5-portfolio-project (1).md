```python
def store_price_data(engine, price_data_dict):
    """Store price data in database"""
    for (symbol, timeframe), df in price_data_dict.items():
        # Select only needed columns
        cols_to_store = ['timestamp', 'symbol', 'timeframe', 'open', 'high', 'low', 'close', 'volume']
        df_to_store = df[cols_to_store]
        
        # Store in database with "upsert" logic (update or insert)
        df_to_store.to_sql('price_data', engine, if_exists='append', index=False,
                          method='multi', chunksize=1000)
        
        print(f"Stored {len(df_to_store)} rows for {symbol} {timeframe} in price_data table")

def store_technical_indicators(engine, indicators_dict):
    """Store technical indicators in database"""
    for (symbol, timeframe), df in indicators_dict.items():
        # Drop rows with NaN values that would cause database errors
        df_clean = df.dropna()
        
        # Store in database with "upsert" logic
        df_clean.to_sql('technical_indicators', engine, if_exists='append', index=False,
                       method='multi', chunksize=1000)
        
        print(f"Stored {len(df_clean)} rows for {symbol} {timeframe} in technical_indicators table")

def store_account_info(engine, account_df):
    """Store account information in database"""
    if account_df is not None and not account_df.empty:
        account_df.to_sql('account_data', engine, if_exists='append', index=False)
        print("Stored account data")

def store_trades_data(engine, trades_df):
    """Store trades data in database"""
    if trades_df is not None and not trades_df.empty:
        trades_df.to_sql('trades_data', engine, if_exists='append', index=False,
                        method='multi', chunksize=1000, 
                        # On conflict with existing tickets, do nothing
                        # (We'll handle updates in a separate function)
                        postgresql_on_conflict='DO NOTHING')
        print(f"Stored {len(trades_df)} trades in trades_data table")
```

### Main Data Extraction Process

```python
def extract_and_store_data():
    """Main function to extract MT5 data and store in database"""
    print(f"Starting data extraction at {dt.datetime.now()}")
    
    # Connect to MT5
    if not connect_to_mt5():
        return
    
    try:
        # Create database engine
        engine = create_db_engine()
        
        # Create database tables if they don't exist
        create_database_tables(engine)
        
        # Define symbols and timeframes to extract
        symbols = ["EURUSD", "GBPUSD", "USDJPY", "AUDUSD", "USDCAD", "XAUUSD"]
        timeframes = [mt5.TIMEFRAME_M15, mt5.TIMEFRAME_H1, mt5.TIMEFRAME_D1]
        
        # Extract price data
        price_data = extract_price_data(symbols, timeframes)
        
        # Calculate technical indicators
        indicators = calculate_technical_indicators(price_data)
        
        # Extract account information
        account_info = extract_account_info()
        
        # Extract trades history
        trades_history = extract_trades_history()
        
        # Extract open positions
        open_positions = extract_open_positions()
        
        # Combine trades history and open positions if both exist
        if trades_history is not None and open_positions is not None:
            trades_data = pd.concat([trades_history, open_positions], ignore_index=True)
        elif trades_history is not None:
            trades_data = trades_history
        else:
            trades_data = open_positions
        
        # Store data in database
        store_price_data(engine, price_data)
        store_technical_indicators(engine, indicators)
        store_account_info(engine, account_info)
        store_trades_data(engine, trades_data)
        
        print(f"Data extraction completed at {dt.datetime.now()}")
    
    except Exception as e:
        print(f"Error extracting data: {e}")
    
    finally:
        # Disconnect from MT5
        disconnect_from_mt5()
```

### Scheduled Data Extraction

```python
def schedule_data_extraction():
    """Set up scheduled data extraction"""
    # Run immediately once
    extract_and_store_data()
    
    # Then schedule regular runs
    # Extract data every hour
    schedule.every().hour.do(extract_and_store_data)
    
    # Also extract data at specific market open times
    schedule.every().monday.at("00:00").do(extract_and_store_data)  # Sydney open
    schedule.every().monday.at("08:00").do(extract_and_store_data)  # London open
    schedule.every().monday.at("13:30").do(extract_and_store_data)  # New York open
    # Repeat for other weekdays...
    
    print("Data extraction scheduled")
    
    # Run the scheduler
    while True:
        schedule.run_pending()
        time.sleep(60)  # Check every minute

if __name__ == "__main__":
    schedule_data_extraction()
```

## Part 2: Database Schema Optimization

### Creating Database Indexes for Performance

Create a file named `create_indexes.sql`:

```sql
-- Create indexes for price_data table
CREATE INDEX IF NOT EXISTS idx_price_data_symbol_timeframe ON price_data(symbol, timeframe);
CREATE INDEX IF NOT EXISTS idx_price_data_timestamp ON price_data(timestamp);

-- Create indexes for technical_indicators table
CREATE INDEX IF NOT EXISTS idx_technical_indicators_symbol_timeframe ON technical_indicators(symbol, timeframe);
CREATE INDEX IF NOT EXISTS idx_technical_indicators_timestamp ON technical_indicators(timestamp);

-- Create indexes for trades_data table
CREATE INDEX IF NOT EXISTS idx_trades_data_symbol ON trades_data(symbol);
CREATE INDEX IF NOT EXISTS idx_trades_data_open_time ON trades_data(open_time);
CREATE INDEX IF NOT EXISTS idx_trades_data_close_time ON trades_data(close_time);
```

### Creating Database Views for Analysis

Create a file named `create_views.sql`:

```sql
-- Create view for daily account balance history
CREATE OR REPLACE VIEW daily_account_balance AS
SELECT 
    DATE_TRUNC('day', timestamp) AS date,
    AVG(balance) AS avg_balance,
    AVG(equity) AS avg_equity,
    MIN(equity) AS min_equity,
    MAX(equity) AS max_equity
FROM account_data
GROUP BY DATE_TRUNC('day', timestamp)
ORDER BY date;

-- Create view for trade performance by symbol
CREATE OR REPLACE VIEW trade_performance_by_symbol AS
SELECT
    symbol,
    COUNT(*) AS total_trades,
    SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END) AS winning_trades,
    SUM(CASE WHEN profit < 0 THEN 1 ELSE 0 END) AS losing_trades,
    SUM(profit) AS total_profit,
    AVG(profit) AS avg_profit_per_trade,
    SUM(CASE WHEN profit > 0 THEN profit ELSE 0 END) / NULLIF(SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END), 0) AS avg_win,
    ABS(SUM(CASE WHEN profit < 0 THEN profit ELSE 0 END)) / NULLIF(SUM(CASE WHEN profit < 0 THEN 1 ELSE 0 END), 0) AS avg_loss,
    (SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END)::FLOAT / NULLIF(COUNT(*), 0) * 100) AS win_rate
FROM trades_data
WHERE close_time IS NOT NULL
GROUP BY symbol
ORDER BY total_profit DESC;

-- Create view for daily trade performance
CREATE OR REPLACE VIEW daily_trade_performance AS
SELECT
    DATE_TRUNC('day', close_time) AS date,
    COUNT(*) AS total_trades,
    SUM(profit) AS daily_profit,
    SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END) AS winning_trades,
    SUM(CASE WHEN profit < 0 THEN 1 ELSE 0 END) AS losing_trades,
    (SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END)::FLOAT / NULLIF(COUNT(*), 0) * 100) AS win_rate
FROM trades_data
WHERE close_time IS NOT NULL
GROUP BY DATE_TRUNC('day', close_time)
ORDER BY date;
```

## Part 3: Power BI Dashboard Creation

### Step 1: Connect Power BI to PostgreSQL Database

1. Open Power BI Desktop
2. Click on "Get Data" > "Database" > "PostgreSQL database"
3. Enter the database connection details:
   - Server: localhost (or your server address)
   - Database: mt5_trading_data
   - Data Connectivity mode: Import
4. Enter your username and password
5. Select the tables and views to import:
   - price_data
   - technical_indicators
   - account_data
   - trades_data
   - daily_account_balance (view)
   - trade_performance_by_symbol (view)
   - daily_trade_performance (view)
6. Click "Load" to import the data

### Step 2: Create Data Relationships

In the Power BI Data Model view:

1. Create a relationship between:
   - price_data and technical_indicators: on symbol, timeframe, timestamp
   - trades_data and price_data: on symbol (many-to-many)
   - account_data to daily_trade_performance: on date

### Step 3: Create Measures for Analysis

Create the following DAX measures:

```
// Cumulative Profit
Cumulative Profit = 
CALCULATE(
    SUM(trades_data[profit]),
    FILTER(
        ALL(trades_data),
        trades_data[close_time] <= MAX(trades_data[close_time])
    )
)

// Equity Curve
Equity Curve = 
VAR CurrentDate = SELECTEDVALUE(Calendar[Date])
RETURN
CALCULATE(
    [Cumulative Profit],
    FILTER(
        ALL(trades_data),
        trades_data[close_time] <= CurrentDate
    )
)

// Drawdown
Drawdown = 
VAR CurrentEquity = [Equity Curve]
VAR MaxEquity = 
    CALCULATE(
        MAX([Equity Curve]),
        FILTER(
            ALL(Calendar),
            Calendar[Date] <= MAX(Calendar[Date])
        )
    )
RETURN
IF(CurrentEquity < MaxEquity, (CurrentEquity - MaxEquity) / MaxEquity, 0)

// Win Rate
Win Rate = 
DIVIDE(
    COUNTROWS(FILTER(trades_data, trades_data[profit] > 0)),
    COUNTROWS(trades_data),
    0
)

// Profit Factor
Profit Factor = 
DIVIDE(
    SUMX(FILTER(trades_data, trades_data[profit] > 0), [profit]),
    ABS(SUMX(FILTER(trades_data, trades_data[profit] < 0), [profit])),
    0
)
```

### Step 4: Create Dashboard Pages

#### Page 1: Account Overview
- Line chart showing Equity Curve over time
- Area chart showing Drawdown percentage
- Cards showing:
  - Total Net Profit
  - Win Rate
  - Profit Factor
  - Maximum Drawdown
- Bar chart showing daily profits/losses

#### Page 2: Trade Analysis
- Table showing trade performance by symbol
- Histogram of trade profits
- Scatter plot of profit vs trade duration
- Pie chart of winning vs losing trades
- Timeline of trades with entry/exit markers

#### Page 3: Market Analysis
- Line charts for price data with technical indicators
- Heatmap showing performance by day of week and hour
- Correlation matrix between traded instruments
- Technical indicator effectiveness analysis

#### Page 4: Risk Management
- Drawdown analysis
- Position sizing analysis
- Risk/reward ratio analysis
- Exposure by instrument over time

### Step 5: Add Interactive Elements
- Date range slicer for filtering all visuals
- Symbol slicer for filtering by trading instrument
- Timeframe slicer for technical analysis
- Drill-through from summary to detailed views

## Part 4: Deployment and Automation

### Running the Data Extraction Script as a Service (Windows)

Create a file named `install_service.bat`:

```batch
@echo off
echo Installing MT5 Data Extraction Service

REM Create a service using NSSM (Non-Sucking Service Manager)
REM Download NSSM from https://nssm.cc/ first

REM Set paths
set PYTHON_PATH=C:\Path\To\Python\python.exe
set SCRIPT_PATH=C:\Path\To\Project\mt5_data_extraction.py
set SERVICE_NAME=MT5DataExtraction

REM Install service
nssm.exe install %SERVICE_NAME% %PYTHON_PATH% %SCRIPT_PATH%
nssm.exe set %SERVICE_NAME% DisplayName "MT5 Data Extraction Service"
nssm.exe set %SERVICE_NAME% Description "Extracts data from MT5 and stores it in PostgreSQL"
nssm.exe set %SERVICE_NAME% Start SERVICE_AUTO_START
nssm.exe set %SERVICE_NAME% AppStdout "C:\Path\To\Project\logs\service.log"
nssm.exe set %SERVICE_NAME% AppStderr "C:\Path\To\Project\logs\error.log"

echo Service installed. Starting service...
net start %SERVICE_NAME%

echo Service started successfully.
```

### Scheduling Power BI Refresh (Linux/macOS)

Create a file named `schedule_powerbi_refresh.sh`:

```bash
#!/bin/bash

# Install Power BI CLI first
# npm install -g powerbi-cli

# Set your Power BI credentials
USERNAME="your_email@example.com"
PASSWORD="your_password"

# Log in to Power BI
powerbi login -u $USERNAME -p $PASSWORD

# Get workspace ID
WORKSPACE_ID=$(powerbi workspace list | grep "MT5 Analytics" | awk '{print $1}')

# Get dataset ID
DATASET_ID=$(powerbi dataset list -w $WORKSPACE_ID | grep "MT5 Trading Data" | awk '{print $1}')

# Refresh the dataset
powerbi dataset refresh -w $WORKSPACE_ID -d $DATASET_ID

echo "Power BI refresh scheduled at $(date)"
```

Add a crontab entry to run this script:
```
0 */6 * * * /path/to/schedule_powerbi_refresh.sh >> /path/to/powerbi_refresh.log 2>&1
```

## Part 5: Documentation and Portfolio Presentation

### Project README.md

```markdown
# MT5 Trading Data Analytics System

## Overview
This project demonstrates a complete end-to-end solution for extracting trading data from MetaTrader 5, storing it in a PostgreSQL database, and visualizing it through interactive Power BI dashboards. It showcases skills in financial data analysis, Python programming, database management, and data visualization.

## Features
- Automated extraction of price data, technical indicators, account information, and trade history from MT5
- Storage in a structured PostgreSQL database with optimized schema
- Interactive Power BI dashboards for trading performance analysis
- Scheduled data extraction and dashboard refreshes

## Technologies Used
- Python with MetaTrader5 library
- PostgreSQL database
- SQLAlchemy ORM
- Power BI for visualization
- Windows Services / Cron for scheduling

## Installation Instructions
1. Install Python requirements: `pip install -r requirements.txt`
2. Set up PostgreSQL database
3. Configure .env file with your MT5 and database credentials
4. Run the data extraction script: `python mt5_data_extraction.py