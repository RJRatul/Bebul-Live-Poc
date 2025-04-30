<template>
  <div class="chart-container" ref="chartRef" style="height: 500px;"></div>
</template>

<script setup>
import { onMounted, ref, onBeforeUnmount } from 'vue'
import { createChart } from 'lightweight-charts'

const chartRef = ref(null)
const chart = ref(null)
const series = ref(null)
const INTERVAL = 60000 // 10-second candles for better visibility
let currentCandle = null
let ws = null

// Enhanced price calculation with spread awareness
function getMidPrice(bid, ask) {
  return (bid + ask) / 2;
}

function getCandleColor(open, close) {
  return close >= open ? '#089981' : '#F23645'
}

function createNewCandle(timestamp, price) {
  return {
    time: Math.floor(timestamp / 1000), // Convert to seconds
    open: price,
    high: price,
    low: price,
    close: price,
  };
}

onMounted(() => {
  chart.value = createChart(chartRef.value, {
    layout: {
      background: { color: '#0E1621' },
      textColor: '#9AA2B1'
    },
    width: chartRef.value.clientWidth,
    height: 500,
    timeScale: {
      timeVisible: true,
      secondsVisible: true,
      borderColor: '#1E293B',
      fixLeftEdge: true,
      fixRightEdge: true
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
    }
  });

  ws = new WebSocket('wss://marketdata.tradermade.com/feedadv');

  ws.onopen = () => {
    ws.send(JSON.stringify({
      userKey: 'wsCgVhfz2ZyK5qn8qUYQ',
      symbol: 'USDJPY'
    }));
  };

  ws.onmessage = (event) => {
    try {
      const tick = JSON.parse(event.data);
      if (!tick?.bid || !tick?.ask) return;

      const price = getMidPrice(tick.bid, tick.ask);
      const timestamp = parseInt(tick.ts);
      const candleTime = Math.floor(timestamp / INTERVAL) * INTERVAL;

      if (!currentCandle || candleTime > currentCandle.originalTime) {
        if (currentCandle) {
          // Finalize previous candle
          series.value.update({
            time: currentCandle.time,
            open: currentCandle.open,
            high: currentCandle.high,
            low: currentCandle.low,
            close: currentCandle.close,
            color: getCandleColor(currentCandle.open, currentCandle.close)
          });
        }

        // Create new candle
        currentCandle = {
          originalTime: candleTime,
          time: candleTime / 1000, // Convert to seconds
          open: price,
          high: price,
          low: price,
          close: price
        };
      } else {
        // Update current candle
        currentCandle.high = Math.max(currentCandle.high, price);
        currentCandle.low = Math.min(currentCandle.low, price);
        currentCandle.close = price;

        // Update series in real-time
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
    for (let entry of entries) {
      chart.value.applyOptions({
        width: entry.contentRect.width,
        height: entry.contentRect.height
      });
    }
  });

  resizeObserver.observe(chartRef.value);
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