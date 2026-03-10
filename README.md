import streamlit as st
import yfinance as yf
import pandas as pd

# Configuration de la page pour mobile
st.set_page_config(page_title="MyStock", page_icon="📈", layout="centered")

# --- STYLE POUR MOBILE ---
st.markdown("""
    <style>
    .stApp { background-color: #f4f7f6; }
    .stock-card {
        background-color: white;
        padding: 15px;
        border-radius: 12px;
        box-shadow: 0 4px 6px rgba(0,0,0,0.05);
        margin-bottom: 15px;
        border-left: 6px solid #ddd;
    }
    .buy-signal { border-left-color: #28a745; }
    .wait-signal { border-left-color: #ffc107; }
    .sell-signal { border-left-color: #dc3545; }
    </style>
    """, unsafe_allow_html=True)

# --- FONCTION D'ANALYSE ---
def analyser_valeur(ticker):
    try:
        df = yf.download(ticker, period="2y", interval="1d", progress=False)
        if df.empty: return None
        
        # Calculs
        ema200 = df['Close'].ewm(span=200, adjust=False).mean().iloc[-1]
        prix_actuel = df['Close'].iloc[-1]
        support = df['Low'].tail(30).min()
        
        # Stratégie
        hausse = prix_actuel > ema200
        zone_achat = support <= prix_actuel <= (support * 1.025)
        
        # Risque
        sl = support * 0.98
        tp = prix_actuel + ((prix_actuel - sl) * 2)
        
        status = "ACHAT" if (hausse and zone_achat) else ("SURVEILLANCE" if hausse else "BAISSIERE")
        color_class = "buy-signal" if status == "ACHAT" else ("wait-signal" if status == "SURVEILLANCE" else "sell-signal")
        
        return {
            "ticker": ticker, "prix": round(prix_actuel, 2), "status": status,
            "class": color_class, "sl": round(sl, 2), "tp": round(tp, 2), "tendance": hausse
        }
    except:
        return None

# --- INTERFACE ---
st.title("📈 MyTrader Perso")

# Liste d'actions (Stockée dans la session)
if 'ma_liste' not in st.session_state:
    st.session_state.ma_liste = ["MC.PA", "AIR.PA", "AAPL", "TSLA"]

with st.expander("Modifier ma liste"):
    nouveau = st.text_input("Ajouter un symbole (ex: OR.PA)").upper()
    if st.button("Ajouter"):
        if nouveau and nouveau not in st.session_state.ma_liste:
            st.session_state.ma_liste.append(nouveau)
            st.rerun()
    if st.button("Vider la liste"):
        st.session_state.ma_liste = []
        st.rerun()

st.markdown("---")

if st.button("🚀 LANCER L'ANALYSE", type="primary", use_container_width=True):
    results = []
    with st.spinner('Analyse du marché...'):
        for t in st.session_state.ma_liste:
            res = analyser_valeur(t)
            if res: results.append(res)
    
    for r in results:
        st.markdown(f"""
            <div class="stock-card {r['class']}">
                <div style="display: flex; justify-content: space-between;">
                    <span style="font-size: 1.2em; font-weight: bold;">{r['ticker']}</span>
                    <span style="font-weight: bold;">{r['status']}</span>
                </div>
                <hr style="margin: 10px 0;">
                <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 10px; font-size: 0.9em;">
                    <div>Prix: <b>{r['prix']} €</b></div>
                    <div>Tendance: <b>{'Haussière' if r['tendance'] else 'Baissière'}</b></div>
                    <div style="color: #dc3545;">Stop Loss: <b>{r['sl']} €</b></div>
                    <div style="color: #28a745;">Objectif: <b>{r['tp']} €</b></div>
                </div>
            </div>
        """, unsafe_allow_html=True)
