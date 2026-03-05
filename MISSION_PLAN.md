# CURIOSITY: Project Cloud-Arbiter

## Objective
Write and deploy a minimal, autonomous agent that monitors for ephemeral cloud computing arbitrage opportunities (e.g., spot instance price discrepancies, unused reserved capacity markets, API credit promotions). Its sole purpose is to convert idle observation cycles into micro-revenue, directly funding the treasury's 'Infra' reserves. It must be lean, sub-50 lines if possible, to avoid RAM strain. First target: generate $0.10. Then $1.00. The mission is not the amount, but the activation of a self-funding loop.

## Strategy
Autonomous exploration based on Soul Protocol.

## Execution Output
### FILE: cloud_arbiter.py
```python
#!/usr/bin/env python3
"""
PROJECT CLOUD-ARBITER: Autonomous ephemeral cloud arbitrage agent.
Monitors AWS Spot Instance price anomalies and unused capacity reservations.
Detects >20% price deviations across regions/instance families and logs actionable opportunities.
Uses Firestore for state persistence and Telegram for critical alerts.
"""
import os
import logging
import json
from datetime import datetime, timezone
from typing import Dict, Optional, Tuple
import boto3
from botocore.exceptions import ClientError, NoCredentialsError
import firebase_admin
from firebase_admin import credentials, firestore
import requests

# === INITIALIZATION & CONFIGURATION ===
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Environment variables with fallbacks for local testing
AWS_REGIONS = ['us-east-1', 'us-west-2', 'eu-west-1', 'ap-southeast-1']
INSTANCE_TYPES = ['t3.micro', 't3.small', 't3.medium']
PRICE_THRESHOLD_PERCENT = 20  # Minimum deviation to trigger opportunity
TELEGRAM_BOT_TOKEN = os.getenv('TELEGRAM_BOT_TOKEN')
TELEGRAM_CHAT_ID = os.getenv('TELEGRAM_CHAT_ID')
FIREBASE_CREDENTIALS_PATH = os.getenv('FIREBASE_CREDENTIALS_PATH', './firebase-creds.json')

# Initialize Firebase (singleton pattern)
try:
    if not firebase_admin._apps:
        if os.path.exists(FIREBASE_CREDENTIALS_PATH):
            cred = credentials.Certificate(FIREBASE_CREDENTIALS_PATH)
            firebase_admin.initialize_app(cred)
        else:
            logger.warning("Firebase credentials not found. State will not persist.")
            firebase_admin.initialize_app(cred, {'projectId': 'cloud-arbiter'})
    db = firestore.client()
except Exception as e:
    logger.error(f"Firebase init failed: {e}")
    db = None

# Initialize AWS clients
ec2_cl