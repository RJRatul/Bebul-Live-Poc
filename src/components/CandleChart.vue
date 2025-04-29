<template>
  <div class="candle-chart-container" ref="chartRef" style="height: 500px;"></div>
</template>

<script setup>
import { onMounted, ref } from 'vue'
import { createChart } from 'lightweight-charts'

const chartRef = ref(null)
const chart = ref(null)
const series = ref(null)

const INTERVAL = 10000 // 10-second candles
const buffer = new Map()
const pendingUpdates = new Map()

function getPrice(tick) {
  return tick.mid || tick.bid || tick.ask || null
}

const updateChart = () => {
  pendingUpdates.forEach((update, bucket) => {
    if (!buffer.has(bucket)) {
      // new candle
      const candle = { ...update, time: Math.floor(bucket / 1000) }
      buffer.set(bucket, candle)
      series.value.update(candle)
    } else {
      const existing = buffer.get(bucket)
      existing.high = Math.max(existing.high, update.high)
      existing.low = Math.min(existing.low, update.low)
      existing.close = update.close
      buffer.set(bucket, existing)
      series.value.update(existing)
    }
  })

  pendingUpdates.clear()
  requestAnimationFrame(updateChart)
}

onMounted(() => {
  chart.value = createChart(chartRef.value, {
    layout: { background: { color: '#000000' }, textColor: '#fff' },
    width: chartRef.value.clientWidth,
    height: 500,
  })

  series.value = chart.value.addCandlestickSeries()
  chart.value.timeScale().fitContent()

  const ws = new WebSocket('wss://marketdata.tradermade.com/feedadv')

  ws.onopen = () => {
    ws.send(JSON.stringify({
      userKey: 'ws72LgyKgNpU0u3Em0Xw',
      symbol: 'EURJPY'
    }))
  }

  ws.onmessage = (event) => {
    try {
      const raw = event.data.trim()
      if (!raw.startsWith('{') && !raw.startsWith('[')) return

      const tick = JSON.parse(raw)
      const price = getPrice(tick)
      const ts = tick.ts
      if (!price || !ts || !tick.symbol) return

      const bucket = Math.floor(ts / INTERVAL) * INTERVAL

      let current = pendingUpdates.get(bucket)
      const existing = buffer.get(bucket)

      if (!current) {
        if (existing) {
          current = { ...existing }
        } else {
          current = { open: price, high: price, low: price, close: price }
        }
      }

      current.high = Math.max(current.high, price)
      current.low = Math.min(current.low, price)
      current.close = price

      // lock the open price once set
      if (!current.open && price) current.open = price

      pendingUpdates.set(bucket, current)
    } catch (err) {
      console.error('Error processing message:', err)
    }
  }

  ws.onclose = () => {
    setTimeout(() => location.reload(), 3000)
  }

  requestAnimationFrame(updateChart)

  setInterval(() => {
    const now = Date.now()
    const cutoff = now - INTERVAL * 60 // Keep last 60 seconds of history
    for (const [bucket] of buffer) {
      if (bucket < cutoff) buffer.delete(bucket)
    }
  }, 1000)
})
</script>


<style>
.candle-chart-container {
  width: 100%;
  transform: translateZ(0);
  will-change: transform;
  backface-visibility: hidden;
}
</style>