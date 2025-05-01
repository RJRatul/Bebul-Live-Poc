<template>
  <div class="chart-container" ref="chartRef" style="height: 500px;"></div>
</template>

<script setup>
import { onMounted, ref, onBeforeUnmount } from 'vue'
import { createChart } from 'lightweight-charts'
import axios from 'axios'

const chartRef = ref(null)
const chart = ref(null)
const series = ref(null)
const INTERVAL = 60000 // 1-minute candles
let currentCandle = null
let ws = null

const auth_liveData = {
  accountId: "53ddc00e-eccd-4973-9eac-2c84f84de986",
  token:
    "eyJhbGciOiJSUzUxMiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI0YmZlZDFlYWIzMjIwNmM0MWM3ZjMzZDI4YTMwNjIyYSIsInBlcm1pc3Npb25zIjpbXSwiYWNjZXNzUnVsZXMiOlt7ImlkIjoidHJhZGluZy1hY2NvdW50LW1hbmFnZW1lbnQtYXBpIiwibWV0aG9kcyI6WyJ0cmFkaW5nLWFjY291bnQtbWFuYWdlbWVudC1hcGk6cmVzdDpwdWJsaWM6KjoqIl0sInJvbGVzIjpbInJlYWRlciIsIndyaXRlciJdLCJyZXNvdXJjZXMiOlsiKjokVVNFUl9JRCQ6KiJdfSx7ImlkIjoibWV0YWFwaS1yZXN0LWFwaSIsIm1ldGhvZHMiOlsibWV0YWFwaS1hcGk6cmVzdDpwdWJsaWM6KjoqIl0sInJvbGVzIjpbInJlYWRlciIsIndyaXRlciJdLCJyZXNvdXJjZXMiOlsiKjokVVNFUl9JRCQ6KiJdfSx7ImlkIjoibWV0YWFwaS1ycGMtYXBpIiwibWV0aG9kcyI6WyJtZXRhYXBpLWFwaTp3czpwdWJsaWM6KjoqIl0sInJvbGVzIjpbInJlYWRlciIsIndyaXRlciJdLCJyZXNvdXJjZXMiOlsiKjokVVNFUl9JRCQ6KiJdfSx7ImlkIjoibWV0YWFwaS1yZWFsLXRpbWUtc3RyZWFtaW5nLWFwaSIsIm1ldGhvZHMiOlsibWV0YWFwaS1hcGk6d3M6cHVibGljOio6KiJdLCJyb2xlcyI6WyJyZWFkZXIiLCJ3cml0ZXIiXSwicmVzb3VyY2VzIjpbIio6JFVTRVJfSUQkOioiXX0seyJpZCI6Im1ldGFzdGF0cy1hcGkiLCJtZXRob2RzIjpbIm1ldGFzdGF0cy1hcGk6cmVzdDpwdWJsaWM6KjoqIl0sInJvbGVzIjpbInJlYWRlciIsIndyaXRlciJdLCJyZXNvdXJjZXMiOlsiKjokVVNFUl9JRCQ6KiJdfSx7ImlkIjoicmlzay1tYW5hZ2VtZW50LWFwaSIsIm1ldGhvZHMiOlsicmlzay1tYW5hZ2VtZW50LWFwaTpyZXN0OnB1YmxpYzoqOioiXSwicm9sZXMiOlsicmVhZGVyIiwid3JpdGVyIl0sInJlc291cmNlcyI6WyIqOiRVU0VSX0lEJDoqIl19LHsiaWQiOiJjb3B5ZmFjdG9yeS1hcGkiLCJtZXRob2RzIjpbImNvcHlmYWN0b3J5LWFwaTpyZXN0OnB1YmxpYzoqOioiXSwicm9sZXMiOlsicmVhZGVyIiwid3JpdGVyIl0sInJlc291cmNlcyI6WyIqOiRVU0VSX0lEJDoqIl19LHsiaWQiOiJtdC1tYW5hZ2VyLWFwaSIsIm1ldGhvZHMiOlsibXQtbWFuYWdlci1hcGk6cmVzdDpkZWFsaW5nOio6KiIsIm10LW1hbmFnZXItYXBpOnJlc3Q6cHVibGljOio6KiJdLCJyb2xlcyI6WyJyZWFkZXIiLCJ3cml0ZXIiXSwicmVzb3VyY2VzIjpbIio6JFVTRVJfSUQkOioiXX0seyJpZCI6ImJpbGxpbmctYXBpIiwibWV0aG9kcyI6WyJiaWxsaW5nLWFwaTpyZXN0OnB1YmxpYzoqOioiXSwicm9sZXMiOlsicmVhZGVyIl0sInJlc291cmNlcyI6WyIqOiRVU0VSX0lEJDoqIl19XSwiaWdub3JlUmF0ZUxpbWl0cyI6ZmFsc2UsInRva2VuSWQiOiIyMDIxMDIxMyIsImltcGVyc29uYXRlZCI6ZmFsc2UsInJlYWxVc2VySWQiOiI0YmZlZDFlYWIzMjIwNmM0MWM3ZjMzZDI4YTMwNjIyYSIsImlhdCI6MTczODkyNTg5M30.E6RrcDK8axUeelmre0CeSoUPrZCHvivpVXnc74pJjooFt64q2oCrW_uqZ54KoDpC3KGx69avLbDAgX48td9DrNSd7C5w9tDk0wynd2cNZRPinKhCor9ATsQ_ppx40D2o28yc1t4EvQ_VLgFE8En-57w7--zPOdf7HthSY0ECmU9-_ae_9p1AJ437h8Gssoq8-44NhZQpvWGxzaB8Vt5SRCuycLJLqE_L7z9ARnH_Ah3kRskGDMqTCGtsTIT_O-EnSqhaZv-FZTagyjCVbfdq5LAGJkvG8XzeI-kHaHwqDvUF20l_f_Nhv7JNod6diAf_TTZv-61-hUAa4udN9l_jbO-GTYkbIjOyNmijSY8NZO_bQ0Pd74ntNlblkCVnnwGZEYlvWxhnkGZjywDAgAlPwdgoL0VJsI9zkfnqAmJGbscZ2GwA9WXLmx1RQrR2FiVT46dZt6KmpP-GKOS1hFKq5Of1wJ9dN7RgfdrV1FHuXcJV4B8QTVDsUBXqEJqNf3DYf0m11Wgn3ynGOKuA_DkbkGcHy1QBBU5f0MO_gPGlCubd_WK6qGo6bQQODHRsAhKN0weS2GkJbHUnKEMUxkzvAF5_AmRMiDEeI9R1gDto-e9lkkddRZEhgTInsMVsz6EXNhPZ8NYf5gOaIlRVMm2_IzOTkyT3wP0ockFSB1ZnbSg", // Your actual token
};

const api_live_data = axios.create({
  baseURL: "https://mt-market-data-client-api-v1.new-york.agiliumtrade.ai",
  headers: {
    "auth-token": auth_liveData.token,
    "Content-Type": "application/json",
    Accept: "application/json",
  },
});

// Helper functions
function getMidPrice(bid, ask) {
  return (bid + ask) / 2;
}

function getCandleColor(open, close) {
  return close >= open ? '#089981' : '#F23645';
}

async function fetchHistoricalMarketData(symbol, timeframe) {
  try {
    const response = await api_live_data.get(
      `/users/current/accounts/${auth_liveData.accountId}/historical-market-data/symbols/USDJPY/timeframes/1m/candles?limit=1000`
    );
    return response.data;
  } catch (error) {
    console.error(`Error fetching historical data for USDJPY:`, error);
    throw error;
  }
}

onMounted(async () => {
  // Initialize chart
  chart.value = createChart(chartRef.value, {
    layout: {
      background: { color: '#0E1621' },
      textColor: '#9AA2B1'
    },
    width: chartRef.value.clientWidth,
    height: 500,
    timeScale: {
      timeVisible: true,
      secondsVisible: false,
      borderColor: '#1E293B',
      fixLeftEdge: true,
    },
    rightPriceScale: {
      borderColor: '#1E293B',
      entireTextOnly: true
    },
    grid: {
      vertLines: { color: '#1E293B' },
      horzLines: { color: '#1E293B' }
    }
  });

  series.value = chart.value.addCandlestickSeries({
    upColor: '#089981',
    downColor: '#F23645',
    borderUpColor: '#089981',
    borderDownColor: '#F23645',
    wickUpColor: '#089981',
    wickDownColor: '#F23645',
    priceFormat: {
      minMove: 0.001,
      precision: 3
    },
    priceLineVisible: true,
    lastValueVisible: true,

  });

  try {
    // Load historical data
    const historicalData = await fetchHistoricalMarketData('USDJPY', '1m');
    
    // Process historical data
    const formattedHistory = historicalData
      .filter(candle => candle.time)
      .map(candle => {
        const timestamp = Date.parse(candle.time);
        return {
          time: Math.floor(timestamp / 1000),
          open: parseFloat(candle.open),
          high: parseFloat(candle.high),
          low: parseFloat(candle.low),
          close: parseFloat(candle.close)
        };
      })
      .sort((a, b) => a.time - b.time);

    // Set all historical data
    series.value.setData(formattedHistory);
    
    // Get last 20 candles for initial view
    const last20Candles = formattedHistory.slice(-20);
    const initialViewStart = last20Candles[0]?.time || Math.floor(Date.now() / 1000 - 20 * 60);
    const initialViewEnd = last20Candles[last20Candles.length - 1]?.time || Math.floor(Date.now() / 1000);
    
    // Set initial visible range
    chart.value.timeScale().setVisibleRange({
      from: initialViewStart,
      to: initialViewEnd
    });

    // Get historical data end time
    const lastHistoricalCandle = formattedHistory[formattedHistory.length - 1];
    const historicalEndTime = lastHistoricalCandle 
      ? lastHistoricalCandle.time * 1000 
      : Date.now();

    // Initialize WebSocket connection
    ws = new WebSocket('wss://marketdata.tradermade.com/feedadv');

    ws.onopen = () => {
      console.log('WebSocket connected');
      ws.send(JSON.stringify({
        userKey: 'wsCgVhfz2ZyK5qn8qUYQ',
        symbol: 'USDJPY'
      }));
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    ws.onclose = (event) => {
      console.log('WebSocket closed:', event);
    };

    ws.onmessage = (event) => {
      try {
        const tick = JSON.parse(event.data);
        if (!tick?.bid || !tick?.ask) return;

        const price = getMidPrice(tick.bid, tick.ask);
        const timestamp = parseInt(tick.ts);
        
        // Ignore invalid or historical data
        if (isNaN(timestamp)) return;
        if (timestamp <= historicalEndTime) return;

        // Calculate candle time aligned with historical data
        const baseTime = historicalEndTime + INTERVAL > timestamp 
          ? historicalEndTime 
          : timestamp;
        const candleTime = Math.floor(baseTime / INTERVAL) * INTERVAL;

        if (!currentCandle || candleTime > currentCandle.originalTime) {
          // Finalize previous candle
          if (currentCandle) {
            series.value.update({
              time: currentCandle.time,
              open: currentCandle.open,
              high: currentCandle.high,
              low: currentCandle.low,
              close: currentCandle.close,
              color: getCandleColor(currentCandle.open, currentCandle.close)
            });
          }

          // Create new candle aligned with historical data
          currentCandle = {
            originalTime: candleTime,
            time: candleTime / 1000,
            open: price,
            high: price,
            low: price,
            close: price
          };

          // Adjust view to show last 20 candles
          chart.value.timeScale().setVisibleRange({
            from: currentCandle.time - 19 * 60, // 20 candles * 60 seconds
            to: currentCandle.time + 60
          });
        } else {
          // Update current candle
          currentCandle.high = Math.max(currentCandle.high, price);
          currentCandle.low = Math.min(currentCandle.low, price);
          currentCandle.close = price;

          series.value.update({
            time: currentCandle.time,
            open: currentCandle.open,
            high: currentCandle.high,
            low: currentCandle.low,
            close: currentCandle.close,
            color: getCandleColor(currentCandle.open, currentCandle.close)
          });
        }
      } catch (err) {
        console.error('Error processing tick:', err);
      }
    };

    // Handle window resize
    const resizeObserver = new ResizeObserver(entries => {
      chart.value.applyOptions({
        width: entries[0].contentRect.width,
        height: entries[0].contentRect.height
      });
    });
    resizeObserver.observe(chartRef.value);

  } catch (error) {
    console.error('Failed to initialize chart:', error);
  }
});

onBeforeUnmount(() => {
  if (ws) ws.close();
  if (chart.value) chart.value.remove();
});
</script>

<style>
.chart-container {
  width: 100%;
  background: #0E1621;
  border-radius: 8px;
  overflow: hidden;
}
</style>