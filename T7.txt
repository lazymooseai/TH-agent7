import streamlit as st
import streamlit.components.v1 as components
import datetime
import pandas as pd
import pytz
import requests
from bs4 import BeautifulSoup

# --- KONFIGURAATIO ---
st.set_page_config(page_title="TH Tuottavuusagentti", page_icon="🚕", layout="centered", initial_sidebar_state="collapsed")
HELSINKI_TZ = pytz.timezone('Europe/Helsinki')

# AUTOMAATTINEN PÄIVITYS (5 min / 300000 ms)
components.html("""
    <script>
        setTimeout(function() {
            window.parent.location.reload();
        }, 300000);
    </script>
""", height=0)

# --- CSS INJEKTIO (Lovable-teema + SUURET FONTIT) ---
st.markdown("""
<style>
    /* Tausta ja perusfontti */
    .stApp { background-color: #0B0E14; color: #E2E8F0; font-family: -apple-system, BlinkMacSystemFont, sans-serif; }
    
    /* Yläpalkki */
    .top-bar { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 25px; }
    .clock-container { display: flex; align-items: baseline; gap: 8px; }
    .time-hours { font-size: 5.0rem; font-weight: 800; color: #FFFFFF; letter-spacing: -3px; line-height: 1; }
    .time-seconds { font-size: 2.0rem; font-weight: 600; color: #4ADE80; }
    
    .weather-widget { background: #151923; padding: 12px 18px; border-radius: 12px; text-align: right; border: 1px solid #2D313E; position: relative; transition: transform 0.2s; }
    .weather-widget:hover { transform: scale(1.05); }
    .weather-temp { font-size: 1.8rem; font-weight: 700; color: #FFFFFF; display: flex; align-items: center; gap: 8px; }
    .weather-desc { font-size: 1.0rem; color: #94A3B8; text-transform: uppercase; letter-spacing: 1px; margin-top: 4px;}

    /* Yleiset korttityylit */
    .section-title { font-size: 1.2rem; font-weight: 800; color: #94A3B8; text-transform: uppercase; margin: 40px 0 15px 0; letter-spacing: 1px; display: flex; align-items: center; gap: 8px;}
    .lv-card { background: #151923; border: 1px solid #2D313E; border-radius: 16px; padding: 20px; margin-bottom: 16px; position: relative; transition: transform 0.2s;}
    .lv-card:hover { transform: scale(1.02); background: #1A1E29; border-color: #4ADE80;}
    .card-link { position: absolute; top: 0; left: 0; height: 100%; width: 100%; z-index: 1; text-decoration: none; }
    
    /* Jackpot & Hälytyspalkki */
    .jackpot-card { border: 2px solid #4A1D24; background: linear-gradient(145deg, #1A131A 0%, #151923 100%); }
    .jackpot-title { color: #F87171; font-size: 1.6rem; font-weight: 800; display: flex; align-items: center; gap: 8px; margin-bottom: 6px; }
    .jackpot-desc { color: #94A3B8; font-size: 1.2rem; }
    
    .alert-banner { background: rgba(248, 113, 113, 0.1); border: 2px solid #F87171; border-radius: 12px; padding: 16px; display: flex; align-items: center; gap: 16px; margin-bottom: 25px; position: relative;}
    .alert-text { color: #F87171; font-weight: 800; font-size: 1.3rem; }

    /* Juna & Laivakortit */
    .item-card { display: flex; align-items: center; justify-content: space-between; }
    .item-icon { font-size: 2.2rem; padding-right: 15px; color: #4ADE80; z-index: 2;}
    .item-icon.delayed { color: #FBBF24; }
    .item-details { flex-grow: 1; z-index: 2;}
    .item-name { font-size: 1.5rem; font-weight: 800; color: #FFFFFF; display: flex; align-items: center; gap: 10px; }
    .item-sub { font-size: 1.1rem; color: #94A3B8; margin-top: 4px; font-weight: 500;}
    .item-time { font-size: 2.4rem; font-weight: 800; color: #4ADE80; text-align: right; z-index: 2;}
    .item-time.delayed { color: #FBBF24; }
    
    .live-badge { background: rgba(74, 222, 128, 0.15); color: #4ADE80; padding: 3px 8px; border-radius: 6px; font-size: 0.8rem; font-weight: 800; letter-spacing: 0.5px; }
    
    /* Napit */
    div[data-testid="stButton"] button { border-radius: 12px; font-size: 1.2rem; font-weight: 700; border: 1px solid #2D313E; background-color: #151923; color: #E2E8F0; padding: 10px 0;}
    div[data-testid="stButton"] button:hover { border-color: #4ADE80; color: #4ADE80; }
</style>
""", unsafe_allow_html=True)

# --- TILANHALLINTA ---
if 'selected_station' not in st.session_state:
    st.session_state.selected_station = "HKI"

if 'alue_data' not in st.session_state:
    st.session_state.alue_data = pd.DataFrame({
        'Alue': ['Lentokenttä', 'Keskusta', 'Jätkäsaari', 'Pasila', 'Kallio'],
        'Perus_Kysyntä': [85, 70, 60, 40, 30]
    })

# --- DATAN HAKU (AGENTIT) ---

@st.cache_data(ttl=86400) # Haetaan kerran päivässä asemien nimet
def get_station_names():
    try:
        res = requests.get("https://rata.digitraffic.fi/api/v1/metadata/stations", timeout=5)
        if res.status_code == 200:
            return {s['stationShortCode']: s['stationName'].replace(" asema", "") for s in res.json()}
    except: pass
    return {}

@st.cache_data(ttl=60)
def fetch_live_trains(station_code):
    station_names = get_station_names()
    url = f"https://rata.digitraffic.fi/api/v1/live-trains/station/{station_code}?arriving_trains=40&departing_trains=0&include_nonstopping=false"
    try:
        res = requests.get(url, timeout=5)
        if res.status_code != 200: return []
        trains = res.json()
        valid_trains = []
        now = datetime.datetime.now(HELSINKI_TZ)
        
        for t in trains:
            if t.get('trainCategory') == 'Long-distance':
                timetable = t.get('timeTableRows', [])
                if not timetable: continue
                
                if timetable[-1].get('stationShortCode') == 'HKI':
                    for row in timetable:
                        if row.get('stationShortCode') == station_code and row.get('type') == 'ARRIVAL':
                            scheduled_utc = datetime.datetime.fromisoformat(row['scheduledTime'].replace('Z', '+00:00'))
                            scheduled_hel = scheduled_utc.astimezone(HELSINKI_TZ)
                            
                            live_estimate = row.get('liveEstimateTime')
                            delay_mins = 0
                            if live_estimate:
                                est_utc = datetime.datetime.fromisoformat(live_estimate.replace('Z', '+00:00'))
                                delay_mins = int((est_utc - scheduled_utc).total_seconds() / 60)
                            
                            if scheduled_hel > now - datetime.timedelta(minutes=5):
                                origin_code = timetable[0].get('stationShortCode', 'N/A')
                                origin_name = station_names.get(origin_code, origin_code)
                                
                                valid_trains.append({
                                    "id": f"{t.get('trainType', '')} {t.get('trainNumber', '')}",
                                    "time": scheduled_hel.strftime('%H:%M'),
                                    "sort_time": scheduled_hel,
                                    "delay": delay_mins,
                                    "origin": origin_name
                                })
                            break
        
        valid_trains.sort(key=lambda x: x['sort_time'])
        return valid_trains[:5]
    except:
        return []

@st.cache_data(ttl=300) # Päivitetään laivadata 5 min välein
def fetch_live_ships():
    ship_database = {
        "MS Finlandia": "Länsiterminaali T2", "Finlandia": "Länsiterminaali T2", 
        "MyStar": "Länsiterminaali T2", "Megastar": "Länsiterminaali T2",
        "Viking XPRS": "Katajanokka", "Silja Serenade": "Olympiaterminaali",
        "Gabriella": "Katajanokka", "Viking Cinderella": "Katajanokka"
    }
    ships = []
    try:
        headers = {'User-Agent': 'Mozilla/5.0'}
        res = requests.get("https://averio.fi/laivat/", headers=headers, timeout=5)
        if res.status_code == 200:
            soup = BeautifulSoup(res.text, 'html.parser')
            for tr in soup.find_all('tr'):
                tds = tr.find_all(['td', 'th'])
                if len(tds) >= 4:
                    time_str = tds[0].text.strip()
                    ship_name = tds[2].text.strip()
                    pax_str = tds[3].text.strip()
                    
                    if "Saapumisaika" in time_str or not ship_name: continue
                    if ship_name in ship_database:
                        only_time = time_str.split(' ')[0] if ' ' in time_str else time_str
                        pax_digits = ''.join(filter(str.isdigit, pax_str))
                        pax_count = pax_digits if pax_digits else "0"
                        ships.append({
                            "name": ship_name,
                            "terminal": ship_database[ship_name],
                            "pax": pax_count,
                            "time": only_time
                        })
    except: pass
    return ships[:3] if ships else []

# --- YLÄPALKKI (Aika ja Sää) ---
now = datetime.datetime.now(HELSINKI_TZ)
time_hm = now.strftime("%H:%M")
time_s = now.strftime("%S")

st.markdown(f"""
<div class="top-bar">
    <div class="clock-container">
        <div class="time-hours">{time_hm}</div>
        <div class="time-seconds">{time_s}</div>
    </div>
    <div class="weather-widget">
        <a href="https://www.ilmatieteenlaitos.fi/sade-ja-pilvialueet?area=etela-suomi" target="_blank" class="card-link"></a>
        <div class="weather-temp">☀️ +4° <span style="font-size: 1.0rem; color: #94A3B8;">🌪️ 19m/s</span></div>
        <div class="weather-desc">SADE ALKAMASSA</div>
    </div>
</div>
""", unsafe_allow_html=True)


# --- 1. JACKPOT-ALUE & HÄLYTYKSET ---
st.markdown("""
<div class="lv-card jackpot-card">
    <a href="https://www.vr.fi/radalla/poikkeustilanteet" target="_blank" class="card-link"></a>
    <div class="jackpot-title">⚡ JACKPOT-ALUE: PASILA</div>
    <div class="jackpot-desc">Erikoistilanne alueella. Klikkaa lukeaksesi häiriötiedotteet.</div>
</div>
""", unsafe_allow_html=True)


# --- 2. SATAMAT (LAIVAT AVERIO.FI) ---
st.markdown('<div class="section-title">⛴️ SATAMAT (LAIVAT)</div>', unsafe_allow_html=True)
live_ships = fetch_live_ships()

if live_ships:
    for ship in live_ships:
        st.markdown(f"""
        <div class="lv-card item-card">
            <a href="https://averio.fi/laivat/" target="_blank" class="card-link"></a>
            <div class="item-icon">🚢</div>
            <div class="item-details">
                <div class="item-name">{ship['name']} <span class="live-badge">🔴 LIVE</span></div>
                <div class="item-sub">Tulossa: ~{ship['pax']} hlö • {ship['terminal']}</div>
            </div>
            <div class="item-time">{ship['time']}</div>
        </div>
        """, unsafe_allow_html=True)
else:
    st.info("Laivadataa ei juuri nyt saatavilla.")


# --- 3. JUNAT (KAUKO) LIVE ---
st.markdown('<div class="section-title">🚆 JUNAT (KAUKO)</div>', unsafe_allow_html=True)

tab1, tab2, tab3 = st.columns(3)
with tab1:
    if st.button("HELSINKI", use_container_width=True, type="primary" if st.session_state.selected_station == "HKI" else "secondary"): st.session_state.selected_station = "HKI"; st.rerun()
with tab2:
    if st.button("PASILA", use_container_width=True, type="primary" if st.session_state.selected_station == "PSL" else "secondary"): st.session_state.selected_station = "PSL"; st.rerun()
with tab3:
    if st.button("TIKKURILA", use_container_width=True, type="primary" if st.session_state.selected_station == "TKL" else "secondary"): st.session_state.selected_station = "TKL"; st.rerun()

urls = {
    "HKI": "https://www.vr.fi/radalla?station=HKI&direction=ARRIVAL&stationFilters=%7B%22trainCategory%22%3A%22Long-distance%22%7D",
    "PSL": "https://www.vr.fi/radalla?station=PSL&direction=ARRIVAL&stationFilters=%7B%22trainCategory%22%3A%22Long-distance%22%7D",
    "TKL": "https://www.vr.fi/radalla?station=TKL&direction=ARRIVAL&stationFilters=%7B%22trainCategory%22%3A%22Long-distance%22%7D"
}
current_url = urls[st.session_state.selected_station]
asemien_nimet = {"HKI": "Helsinkiin", "PSL": "Pasilaan", "TKL": "Tikkurilaan"}

trains = fetch_live_trains(st.session_state.selected_station)

if trains:
    for t in trains:
        is_delayed = t['delay'] > 2
        icon_class = "delayed" if is_delayed else ""
        time_class = "delayed" if is_delayed else ""
        sub_text = f"<span style='color: #FBBF24;'>Myöhässä +{t['delay']} min</span>" if is_delayed else "Aikataulussa"
        
        st.markdown(f"""
        <div class="lv-card item-card">
            <a href="{current_url}" target="_blank" class="card-link"></a>
            <div class="item-icon {icon_class}">🚆</div>
            <div class="item-details">
                <div class="item-name">{t['id']} ({t['origin']} ➡️)</div>
                <div class="item-sub">{sub_text} • Saapuu {asemien_nimet[st.session_state.selected_station]}</div>
            </div>
            <div class="item-time {time_class}">{t['time']}</div>
        </div>
        """, unsafe_allow_html=True)
else:
    st.info(f"Ei saapuvia kaukojunia asemalle {asemien_nimet[st.session_state.selected_station]} lähiaikoina.")


# --- 4. TAPAHTUMAT TÄNÄÄN ---
st.markdown('<div class="section-title">🎫 TAPAHTUMAT TÄNÄÄN</div>', unsafe_allow_html=True)

st.markdown("""
<div class="lv-card">
    <a href="https://messukeskus.com/kavijalle/tapahtumat/tapahtumakalenteri" target="_blank" class="card-link"></a>
    <div style="display: flex; justify-content: space-between; margin-bottom: 10px;">
        <div style="z-index: 2;">
            <div style="color: #94A3B8; font-size: 1.0rem; text-transform: uppercase;">Messukeskus</div>
            <div style="font-size: 1.6rem; font-weight: 800; color: #FFFFFF;">Kevätmessut päättyy</div>
        </div>
        <div style="font-size: 2.2rem; font-weight: 800; color: #FBBF24; z-index: 2;">18:00</div>
    </div>
    <div style="background: rgba(148, 163, 184, 0.1); padding: 4px 8px; border-radius: 4px; display: inline-block; font-size: 0.9rem; color: #94A3B8; margin-bottom: 5px; z-index: 2;">SUURI TAPAHTUMA</div>
</div>
""", unsafe_allow_html=True)


# --- 5. KAMERAT - LIIKENNETILANNE ---
st.markdown('<div class="section-title">📷 KAMERAT - LIIKENNETILANNE</div>', unsafe_allow_html=True)
cam1, cam2 = st.columns(2)
with cam1:
    st.image("https://weathercam.digitraffic.fi/C0450701.jpg", caption="Kehä I Leppävaara")
    st.image("https://weathercam.digitraffic.fi/C0150701.jpg", caption="Hakamäentie / Pasila")
with cam2:
    st.image("https://weathercam.digitraffic.fi/C0150201.jpg", caption="Länsiväylä (Lauttasaari)")
    st.image("https://weathercam.digitraffic.fi/C0450401.jpg", caption="Tuusulanväylä (Käpylä)")

st.markdown("<br><br><br>", unsafe_allow_html=True)
