import streamlit as st
import yfinance as yf
import pandas as pd
import numpy as np
import plotly.graph_objects as go

st.set_page_config(page_title="Simulatore PAC ETF", layout="centered")

# ISIN → Ticker mapping (puoi aggiungerne quanti vuoi)
isin_to_ticker = {
    "IE00B4L5Y983": "IWDA.AS",
    "IE00BKM4GZ66": "EIMI.AS",
    "IE00BDBRDM35": "AGGH.AS"
}

st.title("📈 Simulatore PAC ETF")

# INPUT UTENTE
monthly = st.number_input("💰 Importo mensile (€)", value=500, min_value=10, step=10)
years = st.number_input("📆 Periodo (anni)", value=10, min_value=1, step=1)

st.markdown("### 📊 ETF da includere nel PAC")
with st.form("etf_form"):
    isin_1 = st.text_input("ETF 1 - ISIN o Ticker", "IE00B4L5Y983")
    alloc_1 = st.number_input("Allocazione % ETF 1", value=60, min_value=0, max_value=100)

    isin_2 = st.text_input("ETF 2 - ISIN o Ticker", "IE00BKM4GZ66")
    alloc_2 = st.number_input("Allocazione % ETF 2", value=30, min_value=0, max_value=100)

    isin_3 = st.text_input("ETF 3 - ISIN o Ticker", "IE00BDBRDM35")
    alloc_3 = st.number_input("Allocazione % ETF 3", value=10, min_value=0, max_value=100)

    submit = st.form_submit_button("▶️ Simula")

if submit:
    total_alloc = alloc_1 + alloc_2 + alloc_3
    if total_alloc != 100:
        st.error("❌ La somma delle allocazioni deve essere 100%")
    else:
        etfs = [
            {"isin": isin_1.strip().upper(), "alloc": alloc_1},
            {"isin": isin_2.strip().upper(), "alloc": alloc_2},
            {"isin": isin_3.strip().upper(), "alloc": alloc_3},
        ]

        # Funzioni principali
        def get_ticker(code):
            return isin_to_ticker.get(code, code)

        def get_price_history(ticker, years):
            df = yf.Ticker(ticker).history(period=f"{years}y", interval="1mo")
            df = df[["Close"]].dropna()
            df.rename(columns={"Close": ticker}, inplace=True)
            return df

        try:
            prices = []
            allocs = []
            tickers = []

            for etf in etfs:
                ticker = get_ticker(etf["isin"])
                allocs.append(etf["alloc"])
                tickers.append(ticker)
                prices.append(get_price_history(ticker, years))

            df = pd.concat(prices, axis=1).dropna()

            total_months = len(df)
            units = np.zeros(len(tickers))
            equity = []
            invested = 0
            peak = 0
            max_dd = 0

            for i in range(total_months):
                invested += monthly
                for j in range(len(tickers)):
                    alloc_amount = monthly * allocs[j] / 100
                    price = df.iloc[i, j]
                    units[j] += alloc_amount / price
                value = sum(units[j] * df.iloc[i, j] for j in range(len(tickers)))
                peak = max(peak, value)
                dd = (peak - value) / peak if peak else 0
                max_dd = max(max_dd, dd)
                equity.append({"date": df.index[i], "value": value})

            equity_df = pd.DataFrame(equity)
            final_value = equity_df["value"].iloc[-1]
            profit = final_value - invested
            profit_pct = profit / invested * 100

            st.success("✅ Simulazione completata con successo!")

            st.markdown(f"""
            - **Totale investito:** €{invested:,.2f}  
            - **Valore finale:** €{final_value:,.2f}  
            - **Guadagno:** €{profit:,.2f} ({profit_pct:.2f}%)  
            - **Drawdown massimo:** {max_dd*100:.2f}%
            """)

            fig = go.Figure()
            fig.add_trace(go.Scatter(
                x=equity_df["date"],
                y=equity_df["value"],
                mode='lines',
                name='Equity Curve',
                line=dict(color='royalblue')
            ))
            fig.update_layout(title="📈 Equity Curve", xaxis_title="Data", yaxis_title="Valore (€)")
            st.plotly_chart(fig, use_container_width=True)

        except Exception as e:
            st.error("⚠️ Errore nel recupero dei dati. Verifica ticker o connessione.")
            st.text(f"Errore tecnico: {e}")
