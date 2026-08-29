Dockerfile
requirements.txt
api/
trading/
.gitignore
.dockerignore
README.md
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field


app = FastAPI(
    title="LEAN Alpaca AI",
    description="API للتحكم في Alpaca Paper Trading",
    version="1.0.0",
)


ALPACA_API_KEY = os.getenv("ALPACA_API_KEY")
ALPACA_API_SECRET = os.getenv("ALPACA_API_SECRET")

ALPACA_BASE_URL = os.getenv(
    "ALPACA_BASE_URL",
    "https://paper-api.alpaca.markets"
).rstrip("/")

MAX_ORDER_QTY = float(
    os.getenv("MAX_ORDER_QTY", "100")
)


def alpaca_request(method: str, endpoint: str, data=None):

    if not ALPACA_API_KEY or not ALPACA_API_SECRET:
        raise HTTPException(
            status_code=500,
            detail="ALPACA_API_KEY أو ALPACA_API_SECRET غير موجود في Railway Variables"
        )

    headers = {
        "APCA-API-KEY-ID": ALPACA_API_KEY,
        "APCA-API-SECRET-KEY": ALPACA_API_SECRET,
        "Content-Type": "application/json",
    }

    try:

        response = requests.request(
            method=method,
            url=f"{ALPACA_BASE_URL}{endpoint}",
            headers=headers,
            json=data,
            timeout=30,
        )

    except requests.RequestException as e:

        raise HTTPException(
            status_code=502,
            detail=f"فشل الاتصال بـ Alpaca: {str(e)}"
        )

    if not response.ok:

        try:
            error = response.json()
        except Exception:
            error = response.text

        raise HTTPException(
            status_code=response.status_code,
            detail=error
        )

    if not response.text:
        return {}

    return response.json()


class OrderRequest(BaseModel):

    symbol: str = Field(
        ...,
        min_length=1,
        max_length=10
    )

    qty: float = Field(
        ...,
        gt=0
    )

    side: str

    type: str = "market"

    time_in_force: str = "day"


@app.get("/")
def root():

    return {
        "status": "online",
        "service": "LEAN Alpaca AI",
        "version": "1.0.0",
        "environment": "paper",
    }


@app.get("/health")
def health():

    return {
        "status": "healthy"
    }


@app.get("/account")
def account():

    return alpaca_request(
        "GET",
        "/v2/account"
    )


@app.get("/positions")
def positions():

    return alpaca_request(
        "GET",
        "/v2/positions"
    )


@app.get("/orders")
def orders():

    return alpaca_request(
        "GET",
        "/v2/orders?status=open&limit=50"
    )


@app.post("/orders")
def create_order(order: OrderRequest):

    symbol = order.symbol.upper()
    side = order.side.lower()
    order_type = order.type.lower()
    tif = order.time_in_force.lower()

    if side not in ["buy", "sell"]:

        raise HTTPException(
            status_code=400,
            detail="side يجب أن يكون buy أو sell"
        )

    if order_type not in [
        "market",
        "limit",
        "stop",
        "stop_limit"
    ]:

        raise HTTPException(
            status_code=400,
            detail="نوع الأمر غير مدعوم"
        )

    if order.qty > MAX_ORDER_QTY:

        raise HTTPException(
            status_code=400,
            detail=f"الكمية تتجاوز الحد الأقصى {MAX_ORDER_QTY}"
        )

    data = {
        "symbol": symbol,
        "qty": str(order.qty),
        "side": side,
        "type": order_type,
        "time_in_force": tif,
    }

    return alpaca_request(
        "POST",
        "/v2/orders",
        data
    )


@app.delete("/orders/{order_id}")
def cancel_order(order_id: str):

    return alpaca_request(
        "DELETE",
        f"/v2/orders/{order_id}"
    )

import os
import requests


class AlpacaClient:

    def __init__(self):

        self.api_key = os.getenv("ALPACA_API_KEY")
        self.api_secret = os.getenv("ALPACA_API_SECRET")

        self.base_url = os.getenv(
            "ALPACA_BASE_URL",
            "https://paper-api.alpaca.markets"
        ).rstrip("/")

        self.headers = {
            "APCA-API-KEY-ID": self.api_key,
            "APCA-API-SECRET-KEY": self.api_secret,
            "Content-Type": "application/json",
        }

    def get_account(self):

        response = requests.get(
            f"{self.base_url}/v2/account",
            headers=self.headers,
            timeout=30,
        )

        response.raise_for_status()

        return response.json()

    def get_positions(self):

        response = requests.get(
            f"{self.base_url}/v2/positions",
            headers=self.headers,
            timeout=30,
        )

        response.raise_for_status()

        return response.json()

    def create_order(
        self,
        symbol,
        qty,
        side,
        order_type="market",
        time_in_force="day",
    ):

        payload = {
            "symbol": symbol,
            "qty": str(qty),
            "side": side,
            "type": order_type,
            "time_in_force": time_in_force,
        }

        response = requests.post(
            f"{self.base_url}/v2/orders",
            headers=self.headers,
            json=payload,
            timeout=30,
        )

        response.raise_for_status()

        return response.json()
import os


MAX_ORDER_QTY = float(
    os.getenv("MAX_ORDER_QTY", "100")
)


def validate_order(
    symbol: str,
    qty: float,
    side: str
):

    if not symbol:
        raise ValueError(
            "يجب تحديد الرمز"
        )

    if qty <= 0:
        raise ValueError(
            "الكمية يجب أن تكون أكبر من صفر"
        )

    if qty > MAX_ORDER_QTY:
        raise ValueError(
            f"الكمية تتجاوز الحد المسموح {MAX_ORDER_QTY}"
        )

    if side.lower() not in ["buy", "sell"]:
        raise ValueError(
            "side يجب أن يكون buy أو sell"
        )

    return True

.git
.github
__pycache__
*.pyc
.env
.venv
venv
README.md
.env
.venv/
venv/
__pycache__/
*.pyc
.DS_Store