#!/usr/bin/env python3
"""
🛒 E-Commerce Price Tracker with Email Alerts
Scrape prices and get notified when prices drop below a target.
"""

import requests
from bs4 import BeautifulSoup
import argparse
import csv
from datetime import datetime
import smtplib
from email.mime.text import MIMEText
import logging
import json
import os

# Configuration
CONFIG = {
    "email": "muhammaddahir8172@gmail.com",
    "smtp_server": "smtp.gmail.com",
    "smtp_port": 587,
    "email_user": "YOUR_GMAIL@gmail.com",  # Change this to your sender email
    "email_password": "YOUR_APP_PASSWORD"  # Generate App Password in Gmail settings
}

class PriceTracker:
    def __init__(self):
        self.session = requests.Session()
        self.session.headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
        }

    def scrape_price(self, url):
        """Scrape product price from URL"""
        try:
            response = self.session.get(url, timeout=10)
            response.raise_for_status()
            soup = BeautifulSoup(response.text, 'html.parser')
            
            # Amazon-specific selectors (add others as needed)
            price = soup.select_one('.a-price-whole')
            if price:
                return float(price.get_text().replace(',', ''))
            return None
        except Exception as e:
            logging.error(f"Scraping failed: {str(e)}")
            return None

    def check_price_drop(self, url, target_price):
        """Check if current price is below target"""
        current_price = self.scrape_price(url)
        if current_price is None:
            return False
        
        return current_price <= target_price

    def send_email(self, product_name, url, current_price, target_price):
        """Send price alert email"""
        try:
            message = MIMEText(
                f"""Price Alert! 🚨
                
                {product_name} is now ${current_price} (Target: ${target_price})
                Buy now: {url}
                """
            )
            message['Subject'] = f"💰 Price Drop Alert: {product_name}"
            message['From'] = CONFIG['email_user']
            message['To'] = CONFIG['email']

            with smtplib.SMTP(CONFIG['smtp_server'], CONFIG['smtp_port']) as server:
                server.starttls()
                server.login(CONFIG['email_user'], CONFIG['email_password'])
                server.send_message(message)
            
            logging.info(f"Email sent to {CONFIG['email']}")
            return True
        except Exception as e:
            logging.error(f"Email failed: {str(e)}")
            return False

def load_products():
    """Load products from JSON config"""
    if os.path.exists('products.json'):
        with open('products.json') as f:
            return json.load(f)
    return []

def main():
    parser = argparse.ArgumentParser(description="🛒 Price Tracker with Email Alerts")
    parser.add_argument("--url", help="Product URL")
    parser.add_argument("--target", type=float, help="Target price")
    parser.add_argument("--name", help="Product name")
    parser.add_argument("--watch", action='store_true', help="Monitor all products in products.json")
    
    args = parser.parse_args()
    tracker = PriceTracker()

    if args.watch:
        products = load_products()
        for product in products:
            if tracker.check_price_drop(product['url'], product['target']):
                tracker.send_email(
                    product['name'],
                    product['url'],
                    tracker.scrape_price(product['url']),
                    product['target']
                )
    elif args.url and args.target and args.name:
        if tracker.check_price_drop(args.url, args.target):
            tracker.send_email(args.name, args.url, tracker.scrape_price(args.url), args.target)
    else:
        print("Error: Missing arguments. Use --watch or provide --url, --target, and --name")

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    main()
