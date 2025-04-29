<template>
  <div class="chart-container" ref="chartRef" style="height: 500px;"></div>
</template>

<script setup>
import { onMounted, ref } from 'vue'
import { createChart } from 'lightweight-charts'

const chartRef = ref(null)
const chart = ref(null)
const series = ref(null)
const INTERVAL = 10000 // 10-second candles

// Professional trading variables
let currentCandle = null
let previousCandle = null
let cumulativeVolume = 0
let vwapNumerator = 0
let typicalPrice = 0

// Quotex-style color determination
function getCandleColor(open, close) {
  return close >= open ? '#089981' : '#F23645'
}

// Professional price selection logic
function getTransactionPrice(bid, ask, prevClose) {
  const spread = ask - bid
  if (prevClose === null) return (bid + ask) / 2
  return prevClose <= bid ? bid : ask
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
      borderColor: '#1E293B'
    },
    rightPriceScale: {
      borderColor: '#1E293B'
    },
    grid: {
      vertLines: { color: '#1E293B' },
      horzLines: { color: '#1E293B' }
    }
  })

  series.value = chart.value.addCandlestickSeries({
    upColor: '#089981',
    downColor: '#F23645',
    borderUpColor: '#089981',
    borderDownColor: '#F23645',
    wickUpColor: '#089981',
    wickDownColor: '#F23645',
  })

  const ws = new WebSocket('wss://marketdata.tradermade.com/feedadv')

  ws.onopen = () => {
    ws.send(JSON.stringify({
      userKey: 'wsCgVhfz2ZyK5qn8qUYQ',
      symbol: 'AUDCAD'
    }))
  }

  ws.onmessage = (event) => {
    try {
      const tick = JSON.parse(event.data)
      console.log("Raw", tick)
      if (!tick?.bid || !tick?.ask) return

      // Professional price selection
      const price = getTransactionPrice(
        tick.bid,
        tick.ask,
        previousCandle?.close || null
      )
      
      // Typical price for VWAP calculation (HLC/3)
      typicalPrice = (tick.bid + tick.ask + price) / 3
      const volume = tick.volume || 1
      
      const currentTime = Math.floor(Date.now() / INTERVAL) * INTERVAL
      const timestamp = currentTime / 1000

      if (!currentCandle || currentCandle.time !== timestamp) {
        // Finalize previous candle
        if (currentCandle) {
          // Calculate final VWAP
          currentCandle.vwap = vwapNumerator / cumulativeVolume
          // Apply Quotex-style closing logic
          currentCandle.close = currentCandle.vwap
          // Update color based on actual price movement
          currentCandle.color = getCandleColor(currentCandle.open, price)
          series.value.update(currentCandle)
          previousCandle = { ...currentCandle }
        }

        // Initialize new candle
        cumulativeVolume = volume
        vwapNumerator = typicalPrice * volume
        currentCandle = {
          time: timestamp,
          open: previousCandle?.close || price,
          high: price,
          low: price,
          close: price,
          vwap: typicalPrice,
          color: '#9AA2B1' // Neutral color during formation
        }
      } else {
        // Update candle values
        currentCandle.high = Math.max(currentCandle.high, price)
        currentCandle.low = Math.min(currentCandle.low, price)
        
        // Accumulate VWAP components
        cumulativeVolume += volume
        vwapNumerator += typicalPrice * volume
        
        // Update intermediate values
        currentCandle.vwap = vwapNumerator / cumulativeVolume
        currentCandle.close = currentCandle.vwap
        currentCandle.color = getCandleColor(currentCandle.open, price)
      }

      // Professional chart update with smooth transition
      series.value.update({
        ...currentCandle,
        color: currentCandle.color
      })

    } catch (err) {
      console.error('Error processing tick:', err)
    }
  }
})
</script>

<style>
.chart-container {
  width: 100%;
  background: #0E1621;
  border-radius: 8px;
  overflow: hidden;
  transform: translateZ(0);
  will-change: transform;
  backface-visibility: hidden;
}
</style>