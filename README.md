# final

import requests
import json
from datetime import datetime, timedelta
import time

def get_crypto_data_with_retry(coin_id, start_date, end_date, max_retries=3):
    url = f"https://api.coingecko.com/api/v3/coins/{coin_id}/market_chart"

    # Convert start_date and end_date to datetime objects
    start_datetime = datetime.combine(start_date, datetime.min.time())
    end_datetime = datetime.combine(end_date, datetime.min.time())

    params = {
        'vs_currency': 'usd',
        'from': int(start_datetime.timestamp()),
        'to': int(end_datetime.timestamp()),
        'interval': 'daily',
    }

    for attempt in range(max_retries):
        response = requests.get(url, params=params)

        print(f"Attempt {attempt + 1}: {response.url}")

        if response.status_code == 200:
            data = response.json()
            prices = data.get('prices', [])
            if not prices:
                print(f"No data available for {coin_id} in the specified date range.")
                return []

            return prices
        elif response.status_code == 422:
            print(f"No data available for {coin_id} in the specified date range.")
            return []
        elif response.status_code == 429:
            print(f"Rate limit exceeded. Retrying in {2**attempt} seconds...")
            time.sleep(2**attempt)
        else:
            print(f"Failed to fetch data for {coin_id}. Status code: {response.status_code}")
            return None

    print(f"Maximum number of retries reached for {coin_id}. Giving up.")
    return None


def update_csv(coin, start_date, end_date):
    file_path = f"{coin}_result.csv"
    try:
        csv_file = open(file_path, 'r')
        lines = csv_file.readlines()
        csv_file.close()
        last_date = datetime.strptime(lines[-1].split(',')[0], "%Y-%m-%d").date()
    except (FileNotFoundError, IndexError):
        last_date = start_date - timedelta(days=1)

    new_lines = []
    for dt, price in get_crypto_data_with_retry(coin, last_date, end_date):
        new_lines.append(','.join([dt.strftime("%Y-%m-%d"), str(price)]) + '\n')

    with open(file_path, 'a') as csv_file:
        csv_file.writelines(new_lines)

def run_strategy(coin, strategy_fn):
    file_path = f"{coin}_result.csv"
    try:
        csv_file = open(file_path, 'r')
        lines = csv_file.readlines()
        csv_file.close()
        prices = [float(line.split(',')[1]) for line in lines[1:]]
    except FileNotFoundError:
        print(f"CSV file not found for {coin}. Run 'update_csv' first.")
        return None

    return strategy_fn(coin, prices)

def mean_reversion_strategy(prices, coin):
    day = 0
    buy = 0
    trade_profit = 0
    total_profit = 0
    first_buy = []

    for price_entry in prices:
        if len(price_entry) < 2:
            # Skip entries without the expected structure
            continue

        current_price_str = price_entry[1]  # Extract the price value from the API response
        if not current_price_str or not current_price_str.replace('.', '').isdigit():
            # Skip invalid or non-numeric price values
            continue

        current_price = float(current_price_str)
        
        if day > 5:
            average_prices = [entry[1] for entry in prices[day-5:day-1] if len(entry) >= 2]
            if not average_prices or any(not price.replace('.', '').isdigit() for price in average_prices):
                # Skip invalid or non-numeric average price values
                continue

            average = sum(map(float, average_prices)) / 5  # Convert all prices to float

            if current_price < average * 0.99 and buy == 0:
                buy = current_price
                buy = round(buy, 2)
                first_buy.append(buy)

            elif current_price > average * 1.01 and buy != 0:
                sell = current_price
                sell = round(sell, 2)
                trade_profit = (sell - buy)
                trade_profit = round(trade_profit, 2)
                total_profit += trade_profit
                buy = 0

            else:
                pass
        day += 1

    result = {
        "Coin": coin,
        "Strategy": "Mean Reversion",
        "First Buy": first_buy[0] if first_buy else None,
        "Total Profit": total_profit,
        "Last Day Signal": "Sell" if buy != 0 else "No Signal"
    }

    return result



def simple_moving_average_strategy(prices, coin):
    day = 0
    buy = 0
    trade_profit = 0
    total_profit = 0
    first_buy = []

    for price in prices:
        if day > 5:
            current_price = float(price)  # Convert the price to float
            average = sum(map(float, prices[day-5:day])) / 6  # Convert all prices to float

            if current_price > average and buy == 0:
                buy = current_price
                buy = round(buy, 2)
                first_buy.append(buy)
            
            elif current_price < average and buy != 0:
                sell = current_price
                sell = round(sell, 2)
                trade_profit = (sell - buy)
                trade_profit = round(trade_profit, 2)
                total_profit += trade_profit
                buy = 0

            else:
                pass
        day += 1

    result = {
        "Coin": coin,
        "Strategy": "Simple Moving Average",
        "First Buy": first_buy[0] if first_buy else None,
        "Total Profit": total_profit,
        "Last Day Signal": "Sell" if buy != 0 else "No Signal"
    }

    return result


def main():
    coins = ["bitcoin-cash", "eos", "litecoin", "ethereum", "bitcoin", "ripple"]
    end_date = datetime.now().date()
    start_date = end_date - timedelta(days=365)
    results = []

    for coin in coins:
        update_csv(coin, start_date, end_date)
        result_mean_reversion = run_strategy(coin, mean_reversion_strategy)
        result_sma = run_strategy(coin, simple_moving_average_strategy)
        results.extend([result_mean_reversion, result_sma])

        last_day_signal = "No Signal"
        if result_mean_reversion and result_mean_reversion["Last Day Signal"] == "Sell" or \
           result_sma and result_sma["Last Day Signal"] == "Sell":
            last_day_signal = "Sell"

        print(f"You should {last_day_signal} {coin} today.")

    json_output = 'results.json'
    with open(json_output, 'w') as json_file:
        json.dump(results, json_file, indent=4)

    if results:
        max_profit_result = max(results, key=lambda x: x["Total Profit"])
        print(f"\nCryptocurrency and Strategy with the Most Profit:\n{max_profit_result}")

    print("Done!")

if __name__ == "__main__":
    main()
