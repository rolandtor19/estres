import streamlit as st
import yfinance as yf
import pandas as pd
import plotly.graph_objects as go
import numpy as np

# --- TU FUNCIÓN EXACTA (Copiada para testear) ---
def rellenar_huecos_intradia(df, tf, ticker_name=""):
    if df.empty: return df, 0, 0.0
    freq_map = {'1m': '1min', '2m': '2min', '5m': '5min', '15m': '15min', '30m': '30min', '60m': '60min', '90m': '90min', '1h': '1h', '4h': '4h'}
    frecuencia = freq_map.get(tf, '1h')
    es_crypto_247 = ticker_name.endswith("-USD")
    len_original = len(df)
    
    # Lógica de reindexado
    if es_crypto_247:
        indice_completo = pd.date_range(start=df.index[0], end=df.index[-1], freq=frecuencia, tz=df.index.tz)
        df_lleno = df.reindex(indice_completo)
    else:
        # Lógica de Sesiones (Mercado Tradicional)
        dias_unicos = df.groupby(df.index.date)
        indices_dia = []
        for fecha, grupo in dias_unicos:
            start_dia = grupo.index.min()
            end_dia = grupo.index.max()
            rango_dia = pd.date_range(start=start_dia, end=end_dia, freq=frecuencia, tz=df.index.tz)
            indices_dia.append(rango_dia)
            
        if not indices_dia: return df, 0, 0.0
        
        lista_series = [pd.Series(i) for i in indices_dia]
        series_concatenada = pd.concat(lista_series)
        indice_hibrido = pd.DatetimeIndex(series_concatenada.values).unique().sort_values()
        
        if df.index.tz is not None and indice_hibrido.tz is None:
             indice_hibrido = indice_hibrido.tz_localize(df.index.tz)
        df_lleno = df.reindex(indice_hibrido)

    # Métricas
    len_final = len(df_lleno)
    n_huecos = len_final - len_original
    pct_huecos = (n_huecos / len_final) * 100 if len_final > 0 else 0.0

    # Relleno (FFILL)
    col_ref = 'Close' if 'Close' in df_lleno.columns else df_lleno.columns[0]
    df_lleno[col_ref] = df_lleno[col_ref].fillna(method='ffill')
    
    # Marcar visualmente dónde hubo reparación (para el test)
    # Creamos una máscara: True si era NaN antes del fill
    mask_reparado = df.reindex(df_lleno.index)[col_ref].isna()
    
    return df_lleno, n_huecos, pct_huecos, mask_reparado

# --- UI DE PRUEBA ---
st.set_page_config(page_title="Stress Test 4H", layout="wide")
st.title("🔥 Cámara de Tortura: Prueba de Estrés 4H")
st.markdown("Verificando integridad de datos y relleno de huecos.")

# Selector de Tortura
activo_test = st.selectbox("Elegir Víctima:", ["SHIB-USD", "NG=F", "GME", "EURUSD=X", "BTC-USD"])
tf_test = st.selectbox("Timeframe:", ["1h", "4h"], index=1)

if st.button("💣 EJECUTAR TEST"):
    with st.spinner("Torturando datos..."):
        # 1. Descarga Cruda (Sin ajustes automáticos para ver el error real)
        df = yf.download(activo_test, period="59d", interval=tf_test, progress=False, auto_adjust=False)
        
        # Limpieza inicial estándar
        if isinstance(df.columns, pd.MultiIndex):
            try: df = df.xs(activo_test, axis=1, level=1)
            except: pass
        
        # Normalización UTC
        if df.index.tz is not None: df.index = df.index.tz_localize(None)

        st.write(f"**Filas Originales:** {len(df)}")
        
        # 2. Aplicar Reparación
        df_reparado, n_fix, pct_fix, mascara = rellenar_huecos_intradia(df, tf_test, activo_test)
        
        # 3. Diagnóstico
        c1, c2, c3 = st.columns(3)
        c1.metric("Filas Finales", len(df_reparado))
        c2.metric("Huecos Reparados", n_fix, delta_color="inverse")
        c3.metric("Daño Estructural", f"{pct_fix:.2f}%")
        
        if pct_fix > 10.0:
            st.error(f"⚠️ ¡CUIDADO! Se inventó más del 10% de la data. El fractal podría ser una alucinación.")
        elif pct_fix > 0:
            st.success(f"✅ Reparación exitosa. Geometría preservada.")
        else:
            st.info("💎 Data perfecta. No se necesitó reparación.")

        # 4. Visualización Forense
        fig = go.Figure()
        
        # Línea base (Reparada)
        fig.add_trace(go.Scatter(
            x=df_reparado.index, y=df_reparado['Close'],
            mode='lines', name='Data Final (Continua)',
            line=dict(color='gray', width=1)
        ))
        
        # Puntos Rojos (Donde hubo huecos)
        precios_inventados = df_reparado[mascara]['Close']
        if not precios_inventados.empty:
            fig.add_trace(go.Scatter(
                x=precios_inventados.index, y=precios_inventados,
                mode='markers', name='Huecos Rellenados (FIX)',
                marker=dict(color='red', size=6, symbol='x')
            ))
            
        fig.update_layout(title=f"Mapa de Cicatrices: {activo_test} ({tf_test})", template="plotly_dark")
        st.plotly_chart(fig, use_container_width=True)
        
        # 5. Tabla de la Verdad (Ver las costuras)
        if n_fix > 0:
            st.subheader("🔍 Inspección de Costuras (Primeros 10 arreglos)")
            st.dataframe(df_reparado[mascara].head(10))
