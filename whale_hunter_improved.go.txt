/*
══════════════════════════════════════════════════════════════
   🐋 Whale Hunter Pro v6.0 - Enhanced Edition
   سیستم رصد نهنگ مادر و اتو ترید با بهبودهای جدید

   ✨ بهبودهای نسخه 6.0:
   - Data Grid مدرن با فیلتر، سورت و جستجو
   - اعتبارسنجی دقیق با پارامترهای پویا
   - پشتیبانی Bybit و OKX فیوچرز
   - نمایش قیمت فعلی با راست کلیک
   - صفحه چارت UT_Bot Alert
   - کنترل موقعیت‌های منقضی
   - UI/UX بهبود یافته

   برای اجرا:
   go run whale_hunter_improved.go
══════════════════════════════════════════════════════════════
*/

package main

import (
	"bytes"
	"crypto/hmac"
	"crypto/sha256"
	"database/sql"
	"encoding/base64"
	"encoding/hex"
	"encoding/json"
	"fmt"

	"io"
	"log"
	"math"
	"net"
	"net/http"
	"sort"
	"strconv"
	"strings"
	"sync"
	"time"

	"github.com/gorilla/websocket"
	"golang.org/x/net/proxy"

	"context"

	"github.com/redis/go-redis/v9"
	_ "modernc.org/sqlite"
)

// ═══════════════════════════════════════════════════════════════
// تنظیمات پارامتریک
// ═══════════════════════════════════════════════════════════════

type Config struct {
	// API Settings
	APISource     string `json:"api_source"`
	AutoSwitchAPI bool   `json:"auto_switch_api"`
	RedisAddr     string `json:"redis_addr"`
	CacheTTL      int    `json:"cache_ttl"`
	ProxyURL      string `json:"proxy_url"`
	SkipTLSVerify bool   `json:"skip_tls_verify"`
	DemoMode      bool   `json:"demo_mode"`
	Port          int    `json:"port"`
	UseExchangeWS bool   `json:"use_exchange_ws"`

	// Whale Settings
	WhaleThreshold float64 `json:"whale_threshold"`
	PumpThreshold  float64 `json:"pump_threshold"`

	// Validation Settings
	ValidationTimes   []int   `json:"validation_times"`
	ValidationWeights []int   `json:"validation_weights"`
	MinPriceChange    float64 `json:"min_price_change"`

	// Indicators
	UseRSI           bool    `json:"use_rsi"`
	RSIPeriod        int     `json:"rsi_period"`
	RSIOverbought    int     `json:"rsi_overbought"`
	RSIOversold      int     `json:"rsi_oversold"`
	UseMACD          bool    `json:"use_macd"`
	MACDFast         int     `json:"macd_fast"`
	MACDSlow         int     `json:"macd_slow"`
	MACDSignal       int     `json:"macd_signal"`
	UseVolume        bool    `json:"use_volume"`
	VolumeMultiplier float64 `json:"volume_multiplier"`

	// Auto Trade
	Exchange    string  `json:"exchange"`
	APIKey      string  `json:"api_key"`
	SecretKey   string  `json:"secret_key"`
	OKXPass     string  `json:"okx_passphrase"`
	PaperTrade  bool    `json:"paper_trade"`
	TradeAmount float64 `json:"trade_amount"`
	Leverage    int     `json:"leverage"`
	StopLoss    float64 `json:"stop_loss"`
	TakeProfit  float64 `json:"take_profit"`
	Commission  float64 `json:"commission"`

	// Risk Management
	MaxDailyTrades       int `json:"max_daily_trades"`
	MaxConsecutiveLosses int `json:"max_consecutive_losses"`
	MinScoreForTrade     int `json:"min_score_for_trade"`
	MaxOpenPositions     int `json:"max_open_positions"`
}

var config = Config{
	APISource:            "bybit",
	AutoSwitchAPI:        false,
	RedisAddr:            "",
	CacheTTL:             3,
	ProxyURL:             "",
	SkipTLSVerify:        false,
	DemoMode:             false,
	Port:                 8081,
	UseExchangeWS:        false,
	WhaleThreshold:       100000,
	PumpThreshold:        3,
	ValidationTimes:      []int{1, 2, 3},
	ValidationWeights:    []int{20, 30, 50},
	MinPriceChange:       0.1,
	UseRSI:               true,
	RSIPeriod:            14,
	RSIOverbought:        70,
	RSIOversold:          30,
	UseMACD:              true,
	MACDFast:             12,
	MACDSlow:             26,
	MACDSignal:           9,
	UseVolume:            true,
	VolumeMultiplier:     2,
	Exchange:             "bybit",
	PaperTrade:           true,
	TradeAmount:          5,
	Leverage:             5,
	StopLoss:             2,
	TakeProfit:           4,
	Commission:           0.05,
	MaxDailyTrades:       4,
	MaxConsecutiveLosses: 4,
	MinScoreForTrade:     70,
	MaxOpenPositions:     4,
}

// ═══════════════════════════════════════════════════════════════
// مدل‌های داده
// ═══════════════════════════════════════════════════════════════

type MarketData struct {
	Symbol       string  `json:"symbol"`
	Price        float64 `json:"price"`
	Change       float64 `json:"change"`
	High         float64 `json:"high"`
	Low          float64 `json:"low"`
	Volume       float64 `json:"volume"`
	MarketCap    float64 `json:"market_cap"`
	Timestamp    string  `json:"timestamp"`
	Source       string  `json:"source"`
	IsFutures    bool    `json:"is_futures"`
	FundingRate  float64 `json:"funding_rate"`
	OpenInterest float64 `json:"open_interest"`
	OIDelta      float64 `json:"oi_delta"`
}

type Whale struct {
	ID              int64   `json:"id"`
	Symbol          string  `json:"symbol"`
	Price           float64 `json:"price"`
	Volume          float64 `json:"volume"`
	ChangePercent   float64 `json:"change_percent"`
	WhaleType       string  `json:"whale_type"`
	IsReal          bool    `json:"is_real"`
	ConfidenceScore float64 `json:"confidence_score"`
	Timestamp       string  `json:"timestamp"`
	Title           string  `json:"title"`
	OpenInterest    float64 `json:"open_interest"`
	OIDelta         float64 `json:"oi_delta"`
}

type Signal struct {
	ID             int64   `json:"id"`
	Symbol         string  `json:"symbol"`
	SignalType     string  `json:"signal_type"`
	EntryPrice     float64 `json:"entry_price"`
	Price1Min      float64 `json:"price_1min"`
	Price2Min      float64 `json:"price_2min"`
	Price4Min      float64 `json:"price_4min"`
	Change1Min     float64 `json:"change_1min"`
	Change2Min     float64 `json:"change_2min"`
	Change4Min     float64 `json:"change_4min"`
	Valid1Min      bool    `json:"valid_1min"`
	Valid2Min      bool    `json:"valid_2min"`
	Valid4Min      bool    `json:"valid_4min"`
	FinalStatus    string  `json:"final_status"`
	Score          int     `json:"score"`
	Volume         float64 `json:"volume"`
	RSI            float64 `json:"rsi"`
	MACD           float64 `json:"macd"`
	MACDSignal     float64 `json:"macd_signal"`
	MACDHistogram  float64 `json:"macd_histogram"`
	Trend          string  `json:"trend"`
	WhaleFlow      string  `json:"whale_flow"`
	Timestamp      string  `json:"timestamp"`
	ValidatedAt    string  `json:"validated_at"`
	Title          string  `json:"title"`
	PressureChange string  `json:"pressure_change"`
	Pattern        string  `json:"pattern"`
}

type Trade struct {
	ID          int64   `json:"id"`
	SignalID    int64   `json:"signal_id"`
	Symbol      string  `json:"symbol"`
	Side        string  `json:"side"`
	EntryPrice  float64 `json:"entry_price"`
	ExitPrice   float64 `json:"exit_price"`
	Amount      float64 `json:"amount"`
	Leverage    int     `json:"leverage"`
	PnL         float64 `json:"pnl"`
	PnLPercent  float64 `json:"pnl_percent"`
	Commission  float64 `json:"commission"`
	NetPnL      float64 `json:"net_pnl"`
	Status      string  `json:"status"`
	StopLoss    float64 `json:"stop_loss"`
	TakeProfit  float64 `json:"take_profit"`
	Exchange    string  `json:"exchange"`
	OpenedAt    string  `json:"opened_at"`
	ClosedAt    string  `json:"closed_at"`
	CloseReason string  `json:"close_reason"`
}

type PumpDump struct {
	ID            int64   `json:"id"`
	Symbol        string  `json:"symbol"`
	Price         float64 `json:"price"`
	PrevPrice     float64 `json:"prev_price"`
	ChangePercent float64 `json:"change_percent"`
	EventType     string  `json:"event_type"`
	Volume        float64 `json:"volume"`
	Timestamp     string  `json:"timestamp"`
}

type WhaleFlow struct {
	Inflow  float64 `json:"inflow"`
	Outflow float64 `json:"outflow"`
	Net     float64 `json:"net"`
}

type AccountInfo struct {
	Success   bool    `json:"success"`
	Exchange  string  `json:"exchange"`
	Name      string  `json:"name"`
	UID       string  `json:"uid"`
	Balance   float64 `json:"balance"`
	Available float64 `json:"available"`
	Locked    float64 `json:"locked"`
	Error     string  `json:"error,omitempty"`
}

type TradeStats struct {
	TotalTrades     int     `json:"total_trades"`
	Wins            int     `json:"wins"`
	Losses          int     `json:"losses"`
	WinRate         float64 `json:"win_rate"`
	TotalPnL        float64 `json:"total_pnl"`
	TotalCommission float64 `json:"total_commission"`
}

type SignalStats struct {
	Valid    int     `json:"valid"`
	Invalid  int     `json:"invalid"`
	Pending  int     `json:"pending"`
	Accuracy float64 `json:"accuracy"`
}

type UTBotAlert struct {
	Symbol    string  `json:"symbol"`
	Type      string  `json:"type"`
	Price     float64 `json:"price"`
	ATR       float64 `json:"atr"`
	Timestamp string  `json:"timestamp"`
}

// ═══════════════════════════════════════════════════════════════
// دیتابیس SQLite
// ═══════════════════════════════════════════════════════════════

var db *sql.DB
var dbMutex sync.Mutex
var redisClient *redis.Client
var redisOnce sync.Once
var ctx = context.Background()

func initDB() {
	var err error
	db, err = sql.Open("sqlite", "./whale_hunter.db")
	if err != nil {
		log.Fatal(err)
	}

	tables := `
	CREATE TABLE IF NOT EXISTS whales (
		id INTEGER PRIMARY KEY,
		symbol TEXT,
		price REAL,
		volume REAL,
		change_percent REAL,
		whale_type TEXT,
		is_real INTEGER DEFAULT 1,
		confidence_score REAL DEFAULT 0,
		timestamp TEXT,
		title TEXT,
		open_interest REAL DEFAULT 0,
		oi_delta REAL DEFAULT 0,
		saved_at TEXT
	);

	CREATE TABLE IF NOT EXISTS signals (
		id INTEGER PRIMARY KEY,
		symbol TEXT,
		signal_type TEXT,
		entry_price REAL,
		price_1min REAL,
		price_2min REAL,
		price_4min REAL,
		change_1min REAL,
		change_2min REAL,
		change_4min REAL,
		valid_1min INTEGER,
		valid_2min INTEGER,
		valid_4min INTEGER,
		final_status TEXT DEFAULT 'pending',
		score INTEGER DEFAULT 0,
		volume REAL,
		rsi REAL,
		macd REAL,
		macd_signal REAL,
		macd_histogram REAL,
		trend TEXT,
		whale_flow TEXT,
		timestamp TEXT,
		validated_at TEXT,
		title TEXT,
		pressure_change TEXT,
		pattern TEXT,
		saved_at TEXT
	);

	CREATE TABLE IF NOT EXISTS trades (
		id INTEGER PRIMARY KEY,
		signal_id INTEGER,
		symbol TEXT,
		side TEXT,
		entry_price REAL,
		exit_price REAL,
		amount REAL,
		leverage INTEGER,
		pnl REAL,
		pnl_percent REAL,
		commission REAL,
		net_pnl REAL,
		status TEXT DEFAULT 'open',
		stop_loss REAL,
		take_profit REAL,
		exchange TEXT,
		opened_at TEXT,
		closed_at TEXT,
		close_reason TEXT,
		saved_at TEXT
	);

	CREATE TABLE IF NOT EXISTS pump_dumps (
		id INTEGER PRIMARY KEY,
		symbol TEXT,
		price REAL,
		prev_price REAL,
		change_percent REAL,
		event_type TEXT,
		volume REAL,
		timestamp TEXT,
		saved_at TEXT
	);

	CREATE TABLE IF NOT EXISTS utbot_alerts (
		id INTEGER PRIMARY KEY AUTOINCREMENT,
		symbol TEXT,
		type TEXT,
		price REAL,
		atr REAL,
		timestamp TEXT,
		saved_at TEXT
	);

	CREATE INDEX IF NOT EXISTS idx_signals_symbol ON signals(symbol);
	CREATE INDEX IF NOT EXISTS idx_trades_status ON trades(status);
	`

	_, err = db.Exec(tables)
	if err != nil {
		log.Fatal(err)
	}

	log.Println("✅ دیتابیس SQLite آماده")
}

func migrateDB() {
	stmts := []string{
		"ALTER TABLE signals ADD COLUMN final_status TEXT DEFAULT 'pending'",
		"ALTER TABLE signals ADD COLUMN score INTEGER DEFAULT 0",
		"ALTER TABLE signals ADD COLUMN price_1min REAL",
		"ALTER TABLE signals ADD COLUMN price_2min REAL",
		"ALTER TABLE signals ADD COLUMN price_4min REAL",
		"ALTER TABLE signals ADD COLUMN change_1min REAL",
		"ALTER TABLE signals ADD COLUMN change_2min REAL",
		"ALTER TABLE signals ADD COLUMN change_4min REAL",
		"ALTER TABLE signals ADD COLUMN valid_1min INTEGER",
		"ALTER TABLE signals ADD COLUMN valid_2min INTEGER",
		"ALTER TABLE signals ADD COLUMN valid_4min INTEGER",
		"ALTER TABLE signals ADD COLUMN validated_at TEXT",
		"ALTER TABLE signals ADD COLUMN title TEXT",
		"ALTER TABLE signals ADD COLUMN pattern TEXT",
		"ALTER TABLE signals ADD COLUMN pressure_change TEXT",
		"ALTER TABLE whales ADD COLUMN open_interest REAL DEFAULT 0",
		"ALTER TABLE whales ADD COLUMN oi_delta REAL DEFAULT 0",
		"ALTER TABLE whales ADD COLUMN title TEXT",
		"ALTER TABLE trades ADD COLUMN close_reason TEXT",
		"CREATE INDEX IF NOT EXISTS idx_signals_status ON signals(final_status)",
	}
	for _, s := range stmts {
		_, err := db.Exec(s)
		if err != nil {
			// نادیده گرفتن خطا اگر ستون وجود داشته باشد
			continue
		}
	}
}
func initRedis() {
	redisOnce.Do(func() {
		if strings.TrimSpace(config.RedisAddr) == "" {
			redisClient = nil
			return
		}
		client := redis.NewClient(&redis.Options{Addr: config.RedisAddr})
		if _, err := client.Ping(ctx).Result(); err != nil {
			redisClient = nil
			return
		}
		redisClient = client
	})
}

// ═══════════════════════════════════════════════════════════════
// دریافت قیمت از API - پشتیبانی فیوچرز
// ═══════════════════════════════════════════════════════════════

var previousPrices = make(map[string]float64)
var previousOI = make(map[string]float64)
var pricesMutex sync.RWMutex

func generateDemoMarketData() []MarketData {
	timestamp := time.Now().Format(time.RFC3339)
	syms := []string{"BTCUSDT", "ETHUSDT"}
	var out []MarketData
	for _, s := range syms {
		prev := previousPrices[s]
		if prev == 0 {
			if s == "BTCUSDT" {
				prev = 50000
			} else {
				prev = 3000
			}
		}
		price := prev * (1 + (float64(time.Now().Second()%5-2) / 1000.0))
		change := (price - prev) / prev * 100
		volume := 800000 + float64(time.Now().Second())*1000
		oi := previousOI[s]
		if oi == 0 {
			oi = 1e6
		}
		oiDelta := ((oi - previousOI[s]) / oi) * 100
		previousPrices[s] = price
		previousOI[s] = oi
		out = append(out, MarketData{
			Symbol:       s,
			Price:        price,
			Change:       change,
			High:         price * 1.01,
			Low:          price * 0.99,
			Volume:       volume,
			Timestamp:    timestamp,
			Source:       "demo",
			IsFutures:    true,
			FundingRate:  0.01,
			OpenInterest: oi,
			OIDelta:      oiDelta,
		})
	}
	return out
}

// ═══════════════════════════════════════════════════════════════
// تشخیص نهنگ و پامپ/دامپ با تحلیل بهتر
// ═══════════════════════════════════════════════════════════════

func detectWhales(data []MarketData) []Whale {
	var whales []Whale
	timestamp := time.Now().Format(time.RFC3339)

	for _, m := range data {
		if m.Volume >= config.WhaleThreshold {
			whaleType := "buy"
			if m.Change < 0 {
				whaleType = "sell"
			}

			title := generateWhaleTitle(m, whaleType)
			pattern := detectPattern(m)
			pressureChange := detectPressureChange(m)

			whale := Whale{
				ID:              time.Now().UnixNano(),
				Symbol:          m.Symbol,
				Price:           m.Price,
				Volume:          m.Volume,
				ChangePercent:   m.Change,
				WhaleType:       whaleType,
				IsReal:          true,
				ConfidenceScore: calculateWhaleConfidence(m),
				Timestamp:       timestamp,
				Title:           title,
				OpenInterest:    m.OpenInterest,
				OIDelta:         m.OIDelta,
			}

			whales = append(whales, whale)
			saveWhale(whale)
			createSignal(whale, pattern, pressureChange)
		}
	}

	return whales
}

func generateWhaleTitle(m MarketData, whaleType string) string {
	direction := "صعودی"
	if whaleType == "sell" {
		direction = "نزولی"
	}

	strength := "متوسط"
	if m.Volume >= 2000000 {
		strength = "قوی"
	} else if m.Volume >= 1000000 {
		strength = "نسبتاً قوی"
	}

	oiStatus := ""
	if m.OIDelta > 5 {
		oiStatus = " - افزایش OI"
	} else if m.OIDelta < -5 {
		oiStatus = " - کاهش OI"
	}

	return fmt.Sprintf("نهنگ %s %s در %s%s", direction, strength, m.Symbol, oiStatus)
}

func detectPattern(m MarketData) string {
	if m.Price >= m.High*0.98 {
		return "نزدیک سقف"
	} else if m.Price <= m.Low*1.02 {
		return "نزدیک کف"
	} else if m.Change > 10 {
		return "پامپ شدید"
	} else if m.Change < -10 {
		return "دامپ شدید"
	}
	if m.Price > m.High && m.OIDelta > 0 {
		return "یخ شکسته صعودی"
	}
	if m.Price < m.Low && m.OIDelta > 0 {
		return "یخ شکسته نزولی"
	}
	return "خنثی"
}

func detectPressureChange(m MarketData) string {
	if m.Change > 5 && m.OIDelta > 3 {
		return "فشار خرید قوی"
	} else if m.Change < -5 && m.OIDelta > 3 {
		return "فشار فروش قوی"
	} else if m.OIDelta > 5 {
		return "افزایش موقعیت‌ها"
	} else if m.OIDelta < -5 {
		return "بسته شدن موقعیت‌ها"
	}
	return "عادی"
}

func calculateWhaleConfidence(m MarketData) float64 {
	score := 50.0

	if m.Volume >= 2000000 {
		score += 25
	} else if m.Volume >= 1000000 {
		score += 15
	} else if m.Volume >= 500000 {
		score += 10
	}

	if math.Abs(m.Change) >= 10 {
		score += 20
	} else if math.Abs(m.Change) >= 5 {
		score += 15
	} else if math.Abs(m.Change) >= 3 {
		score += 10
	}

	if m.Price >= m.High*0.98 || m.Price <= m.Low*1.02 {
		score += 10
	}

	if math.Abs(m.OIDelta) > 5 {
		score += 10
	}

	return math.Min(score, 100)
}

func detectPumpDumps(data []MarketData) []PumpDump {
	var pumpDumps []PumpDump
	timestamp := time.Now().Format(time.RFC3339)

	pricesMutex.Lock()
	defer pricesMutex.Unlock()

	for _, m := range data {
		if prev, exists := previousPrices[m.Symbol]; exists {
			change := ((m.Price - prev) / prev) * 100

			if math.Abs(change) >= config.PumpThreshold {
				eventType := "pump"
				if change < 0 {
					eventType = "dump"
				}

				pd := PumpDump{
					ID:            time.Now().UnixNano(),
					Symbol:        m.Symbol,
					Price:         m.Price,
					PrevPrice:     prev,
					ChangePercent: change,
					EventType:     eventType,
					Volume:        m.Volume,
					Timestamp:     timestamp,
				}

				pumpDumps = append(pumpDumps, pd)
				savePumpDump(pd)
				createEventSignal(m, eventType)
			}
		}

		previousPrices[m.Symbol] = m.Price
	}

	return pumpDumps
}

func createEventSignal(m MarketData, eventType string) {
	signalType := "LONG"
	if eventType == "dump" {
		signalType = "SHORT"
	}
	trend := "neutral"
	whaleFlow := "neutral"
	if m.Change > 2 {
		trend = "bullish"
		whaleFlow = "inflow"
	} else if m.Change < -2 {
		trend = "bearish"
		whaleFlow = "outflow"
	}
	title := fmt.Sprintf("سیگنال %s - %s (%s)", signalType, m.Symbol, eventType)
	signal := Signal{
		ID:             time.Now().UnixNano(),
		Symbol:         m.Symbol,
		SignalType:     signalType,
		EntryPrice:     m.Price,
		Volume:         m.Volume,
		FinalStatus:    "pending",
		Trend:          trend,
		WhaleFlow:      whaleFlow,
		Timestamp:      time.Now().Format(time.RFC3339),
		Title:          title,
		Pattern:        "رویداد لحظه‌ای",
		PressureChange: "",
	}
	saveSignal(signal)
}

// ═══════════════════════════════════════════════════════════════
// سیگنال‌ها و اعتبارسنجی بهبود یافته
// ═══════════════════════════════════════════════════════════════

func createSignal(whale Whale, pattern string, pressureChange string) {
	signalType := "LONG"
	if whale.WhaleType == "sell" {
		signalType = "SHORT"
	}

	trend := "neutral"
	whaleFlow := "neutral"
	if whale.ChangePercent > 2 {
		trend = "bullish"
		whaleFlow = "inflow"
	} else if whale.ChangePercent < -2 {
		trend = "bearish"
		whaleFlow = "outflow"
	}

	// تولید عنوان برای سیگنال
	title := fmt.Sprintf("سیگنال %s - %s", signalType, whale.Symbol)

	signal := Signal{
		ID:             time.Now().UnixNano(),
		Symbol:         whale.Symbol,
		SignalType:     signalType,
		EntryPrice:     whale.Price,
		Volume:         whale.Volume,
		FinalStatus:    "pending",
		Trend:          trend,
		WhaleFlow:      whaleFlow,
		Timestamp:      time.Now().Format(time.RFC3339),
		Title:          title,
		Pattern:        pattern,
		PressureChange: pressureChange,
	}

	saveSignal(signal)
}

func validateSignal(signal *Signal, currentPrice float64, stage int) bool {
	priceChange := ((currentPrice - signal.EntryPrice) / signal.EntryPrice) * 100
	minChange := config.MinPriceChange

	isValid := false

	if signal.SignalType == "LONG" {
		if priceChange >= minChange {
			isValid = true
		}
	} else {
		if priceChange <= -minChange {
			isValid = true
		}
	}

	// بروزرسانی Title بر اساس اعتبارسنجی
	if isValid {
		signal.Title = fmt.Sprintf("✅ %s - معتبر مرحله %d", signal.Symbol, stage)
	} else {
		signal.Title = fmt.Sprintf("⏳ %s - در حال بررسی", signal.Symbol)
	}

	switch stage {
	case 1:
		signal.Price1Min = currentPrice
		signal.Change1Min = priceChange
		signal.Valid1Min = isValid
	case 2:
		signal.Price2Min = currentPrice
		signal.Change2Min = priceChange
		signal.Valid2Min = isValid
	case 3:
		signal.Price4Min = currentPrice
		signal.Change4Min = priceChange
		signal.Valid4Min = isValid
	}

	return isValid
}

func calculateSignalScore(signal *Signal) int {
	score := 0
	weights := config.ValidationWeights

	if signal.Valid1Min {
		score += weights[0]
	}
	if signal.Valid2Min {
		score += weights[1]
	}
	if signal.Valid4Min {
		score += weights[2]
	}

	if signal.SignalType == "LONG" && signal.Trend == "bullish" {
		score += 10
	} else if signal.SignalType == "SHORT" && signal.Trend == "bearish" {
		score += 10
	}

	if signal.SignalType == "LONG" && signal.WhaleFlow == "inflow" {
		score += 10
	} else if signal.SignalType == "SHORT" && signal.WhaleFlow == "outflow" {
		score += 10
	}

	if score > 100 {
		score = 100
	}

	return score
}

func getFinalStatus(signal *Signal) string {
	validCount := 0
	if signal.Valid1Min {
		validCount++
	}
	if signal.Valid2Min {
		validCount++
	}
	if signal.Valid4Min {
		validCount++
	}

	if validCount >= 2 {
		return "valid"
	}
	return "invalid"
}

func checkPendingSignals(data []MarketData) {
	priceMap := make(map[string]float64)
	for _, m := range data {
		priceMap[m.Symbol] = m.Price
	}

	signals := getPendingSignals()
	now := time.Now()
	validationTimes := config.ValidationTimes

	for _, signal := range signals {
		signalTime, _ := time.Parse(time.RFC3339, signal.Timestamp)
		elapsed := now.Sub(signalTime).Minutes()

		currentPrice, exists := priceMap[signal.Symbol]
		if !exists {
			continue
		}

		updated := false

		if elapsed >= float64(validationTimes[0]) && signal.Price1Min == 0 {
			validateSignal(&signal, currentPrice, 1)
			updated = true
		}

		if elapsed >= float64(validationTimes[1]) && signal.Price2Min == 0 {
			validateSignal(&signal, currentPrice, 2)
			updated = true
		}

		if elapsed >= float64(validationTimes[2]) && signal.Price4Min == 0 {
			validateSignal(&signal, currentPrice, 3)
			signal.FinalStatus = getFinalStatus(&signal)
			signal.Score = calculateSignalScore(&signal)
			signal.ValidatedAt = time.Now().Format(time.RFC3339)

			// بروزرسانی نهایی Title
			if signal.FinalStatus == "valid" {
				signal.Title = fmt.Sprintf("✅ سیگنال معتبر - %s امتیاز: %d", signal.Symbol, signal.Score)
			} else {
				signal.Title = fmt.Sprintf("❌ سیگنال نامعتبر - %s", signal.Symbol)
			}

			updated = true
		}

		if updated {
			updateSignal(signal)

			// ارسال سریع به اتوترید
			if signal.FinalStatus == "valid" && signal.Score >= config.MinScoreForTrade {
				if autoTrader.IsRunning {
					go autoTrader.processValidSignal(signal)
				}
			}
		}
	}
}

// ═══════════════════════════════════════════════════════════════
// اتو ترید با کنترل موقعیت‌های منقضی
// ═══════════════════════════════════════════════════════════════

type AutoTrader struct {
	IsRunning         bool
	DailyTrades       int
	ConsecutiveLosses int
	LastTradeDate     string
	PnL               float64
	TotalCommission   float64
	OpenTrades        map[int64]Trade
	mutex             sync.Mutex
}

var autoTrader = &AutoTrader{
	OpenTrades: make(map[int64]Trade),
}

func (at *AutoTrader) Start() {
	at.mutex.Lock()
	defer at.mutex.Unlock()

	if at.IsRunning {
		return
	}

	at.IsRunning = true
	log.Println("🤖 اتو ترید شروع شد")

	go at.run()
}

func (at *AutoTrader) Stop() {
	at.mutex.Lock()
	defer at.mutex.Unlock()

	at.IsRunning = false
	log.Println("⏹️ اتو ترید متوقف شد")
}

func (at *AutoTrader) run() {
	for at.IsRunning {
		if !at.canTrade() {
			time.Sleep(10 * time.Second)
			continue
		}

		// چک موقعیت‌های باز
		at.checkOpenTrades()

		// پاکسازی موقعیت‌های منقضی
		at.cleanupExpiredPositions()

		time.Sleep(10 * time.Second)
	}
}

func (at *AutoTrader) processValidSignal(signal Signal) {
	at.mutex.Lock()
	defer at.mutex.Unlock()

	// چک محدودیت موقعیت‌های باز
	if len(at.OpenTrades) >= config.MaxOpenPositions {
		log.Printf("⚠️ حداکثر %d موقعیت باز - سیگنال %s نادیده گرفته شد", config.MaxOpenPositions, signal.Symbol)
		return
	}

	// چک که قبلاً ترید نزده باشیم
	if _, exists := at.OpenTrades[signal.ID]; !exists {
		at.executeTrade(signal)
	}
}

func (at *AutoTrader) canTrade() bool {
	today := time.Now().Format("2006-01-02")

	if at.LastTradeDate != today {
		at.DailyTrades = 0
		at.LastTradeDate = today
	}

	if at.DailyTrades >= config.MaxDailyTrades {
		log.Println("⚠️ حداکثر ترید روزانه")
		return false
	}

	if at.ConsecutiveLosses >= config.MaxConsecutiveLosses {
		log.Println("⚠️ ضررهای متوالی - توقف")
		at.Stop()
		return false
	}

	return true
}

func (at *AutoTrader) executeTrade(signal Signal) {
	var stopLoss, takeProfit float64
	if signal.SignalType == "LONG" {
		stopLoss = signal.EntryPrice * (1 - config.StopLoss/100)
		takeProfit = signal.EntryPrice * (1 + config.TakeProfit/100)
	} else {
		stopLoss = signal.EntryPrice * (1 + config.StopLoss/100)
		takeProfit = signal.EntryPrice * (1 - config.TakeProfit/100)
	}

	trade := Trade{
		ID:         time.Now().UnixNano(),
		SignalID:   signal.ID,
		Symbol:     signal.Symbol,
		Side:       signal.SignalType,
		EntryPrice: signal.EntryPrice,
		Amount:     config.TradeAmount,
		Leverage:   config.Leverage,
		StopLoss:   stopLoss,
		TakeProfit: takeProfit,
		Exchange:   config.Exchange,
		Status:     "open",
		OpenedAt:   time.Now().Format(time.RFC3339),
	}

	saveTrade(trade)
	at.OpenTrades[signal.ID] = trade
	at.DailyTrades++

	log.Printf("✅ ترید باز شد: %s %s @ $%.4f", signal.Symbol, signal.SignalType, signal.EntryPrice)

	if config.APIKey != "" && config.SecretKey != "" {
		placeOrder(trade)
	}
}

func (at *AutoTrader) checkOpenTrades() {
	if len(at.OpenTrades) == 0 {
		return
	}

	source := config.Exchange
	if config.Exchange == "okx" {
		source = "okx"
	}

	data, err := fetchMarketDataCached(source)
	if err != nil {
		return
	}

	priceMap := make(map[string]float64)
	for _, m := range data {
		priceMap[m.Symbol] = m.Price
	}

	for signalID, trade := range at.OpenTrades {
		currentPrice, exists := priceMap[trade.Symbol]
		if !exists {
			continue
		}

		shouldClose := false
		closeReason := ""

		if trade.Side == "LONG" {
			if currentPrice <= trade.StopLoss {
				shouldClose = true
				closeReason = "Stop Loss"
			} else if currentPrice >= trade.TakeProfit {
				shouldClose = true
				closeReason = "Take Profit"
			}
		} else {
			if currentPrice >= trade.StopLoss {
				shouldClose = true
				closeReason = "Stop Loss"
			} else if currentPrice <= trade.TakeProfit {
				shouldClose = true
				closeReason = "Take Profit"
			}
		}

		if shouldClose {
			at.closeTrade(signalID, trade, currentPrice, closeReason)
		}
	}
}

func (at *AutoTrader) cleanupExpiredPositions() {
	if len(at.OpenTrades) == 0 {
		return
	}

	now := time.Now()

	for signalID, trade := range at.OpenTrades {
		openTime, err := time.Parse(time.RFC3339, trade.OpenedAt)
		if err != nil {
			continue
		}

		// موقعیت‌هایی که بیش از 24 ساعت باز هستند
		if now.Sub(openTime).Hours() > 24 {
			log.Printf("⚠️ موقعیت منقضی: %s - بسته می‌شود", trade.Symbol)

			// دریافت قیمت فعلی و بستن
			source := config.Exchange
			if config.Exchange == "okx" {
				source = "okx"
			}

			data, err := fetchMarketDataCached(source)
			if err == nil {
				for _, m := range data {
					if m.Symbol == trade.Symbol {
						at.closeTrade(signalID, trade, m.Price, "Expired")
						break
					}
				}
			}
		}
	}
}

func (at *AutoTrader) closeTrade(signalID int64, trade Trade, exitPrice float64, reason string) {
	var pnl float64
	if trade.Side == "LONG" {
		pnl = (exitPrice - trade.EntryPrice) / trade.EntryPrice * trade.Amount * float64(trade.Leverage)
	} else {
		pnl = (trade.EntryPrice - exitPrice) / trade.EntryPrice * trade.Amount * float64(trade.Leverage)
	}

	commission := trade.Amount * config.Commission / 100 * 2
	netPnl := pnl - commission

	trade.ExitPrice = exitPrice
	trade.PnL = pnl
	trade.PnLPercent = (exitPrice - trade.EntryPrice) / trade.EntryPrice * 100
	trade.Commission = commission
	trade.NetPnL = netPnl
	trade.Status = "closed"
	trade.ClosedAt = time.Now().Format(time.RFC3339)
	trade.CloseReason = reason

	updateTrade(trade)

	at.PnL += netPnl
	at.TotalCommission += commission

	if netPnl < 0 {
		at.ConsecutiveLosses++
	} else {
		at.ConsecutiveLosses = 0
	}

	delete(at.OpenTrades, signalID)

	emoji := "✅"
	if netPnl < 0 {
		emoji = "❌"
	}
	log.Printf("%s ترید بسته شد: %s %s | PnL: $%.2f", emoji, trade.Symbol, reason, netPnl)
}

// ═══════════════════════════════════════════════════════════════
// توابع دیتابیس
// ═══════════════════════════════════════════════════════════════

func saveWhale(whale Whale) {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	_, err := db.Exec(`
		INSERT INTO whales (id, symbol, price, volume, change_percent, whale_type, 
		                    is_real, confidence_score, timestamp, title, 
		                    open_interest, oi_delta, saved_at)
		VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
		whale.ID, whale.Symbol, whale.Price, whale.Volume, whale.ChangePercent,
		whale.WhaleType, whale.IsReal, whale.ConfidenceScore, whale.Timestamp,
		whale.Title, whale.OpenInterest, whale.OIDelta,
		time.Now().Format(time.RFC3339))

	if err != nil {
		log.Printf("خطا در ذخیره نهنگ: %v", err)
	}
}

func saveSignal(signal Signal) {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	_, err := db.Exec(`
		INSERT INTO signals (id, symbol, signal_type, entry_price, volume,
		                     trend, whale_flow, timestamp, title, pattern,
		                     pressure_change, saved_at, final_status, score)
		VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'pending', 0)`,
		signal.ID, signal.Symbol, signal.SignalType, signal.EntryPrice,
		signal.Volume, signal.Trend, signal.WhaleFlow, signal.Timestamp,
		signal.Title, signal.Pattern, signal.PressureChange,
		time.Now().Format(time.RFC3339))

	if err != nil {
		log.Printf("خطا در ذخیره سیگنال: %v", err)
	}
}

func updateSignal(signal Signal) {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	_, err := db.Exec(`
		UPDATE signals SET 
			price_1min = ?, price_2min = ?, price_4min = ?,
			change_1min = ?, change_2min = ?, change_4min = ?,
			valid_1min = ?, valid_2min = ?, valid_4min = ?,
			final_status = ?, score = ?, validated_at = ?, title = ?
		WHERE id = ?`,
		signal.Price1Min, signal.Price2Min, signal.Price4Min,
		signal.Change1Min, signal.Change2Min, signal.Change4Min,
		signal.Valid1Min, signal.Valid2Min, signal.Valid4Min,
		signal.FinalStatus, signal.Score, signal.ValidatedAt, signal.Title, signal.ID)

	if err != nil {
		log.Printf("خطا در بروزرسانی سیگنال: %v", err)
	}
}

func getPendingSignals() []Signal {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	rows, err := db.Query(`
		SELECT id, symbol, signal_type, entry_price, 
		       COALESCE(price_1min, 0), COALESCE(price_2min, 0), COALESCE(price_4min, 0),
		       COALESCE(change_1min, 0), COALESCE(change_2min, 0), COALESCE(change_4min, 0),
		       COALESCE(valid_1min, 0), COALESCE(valid_2min, 0), COALESCE(valid_4min, 0),
		       final_status, score, COALESCE(volume, 0), 
		       COALESCE(trend, ''), COALESCE(whale_flow, ''), timestamp,
		       COALESCE(title, ''), COALESCE(pattern, ''), COALESCE(pressure_change, '')
		FROM signals WHERE final_status = 'pending' ORDER BY timestamp ASC`)
	if err != nil {
		return nil
	}
	defer rows.Close()

	var signals []Signal
	for rows.Next() {
		var s Signal
		var v1, v2, v4 int
		err := rows.Scan(&s.ID, &s.Symbol, &s.SignalType, &s.EntryPrice,
			&s.Price1Min, &s.Price2Min, &s.Price4Min,
			&s.Change1Min, &s.Change2Min, &s.Change4Min,
			&v1, &v2, &v4,
			&s.FinalStatus, &s.Score, &s.Volume,
			&s.Trend, &s.WhaleFlow, &s.Timestamp,
			&s.Title, &s.Pattern, &s.PressureChange)
		if err != nil {
			continue
		}
		s.Valid1Min = v1 != 0
		s.Valid2Min = v2 != 0
		s.Valid4Min = v4 != 0
		signals = append(signals, s)
	}
	return signals
}

func getValidSignalsForTrade(minScore int) []Signal {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	rows, err := db.Query(`
		SELECT id, symbol, signal_type, entry_price, score, timestamp, title
		FROM signals 
		WHERE final_status = 'valid' AND score >= ?
		ORDER BY score DESC LIMIT 20`, minScore)
	if err != nil {
		return nil
	}
	defer rows.Close()

	var signals []Signal
	for rows.Next() {
		var s Signal
		rows.Scan(&s.ID, &s.Symbol, &s.SignalType, &s.EntryPrice, &s.Score, &s.Timestamp, &s.Title)
		signals = append(signals, s)
	}
	return signals
}

func getSignals(status string, limit int) []Signal {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	var query string
	var rows *sql.Rows
	var err error

	signals := []Signal{}
	if status == "all" || status == "" {
		query = fmt.Sprintf(`SELECT id, symbol, signal_type, entry_price, 
		         COALESCE(price_1min, 0), COALESCE(price_2min, 0), COALESCE(price_4min, 0),
		         COALESCE(change_1min, 0), COALESCE(change_2min, 0), COALESCE(change_4min, 0),
		         COALESCE(valid_1min, 0), COALESCE(valid_2min, 0), COALESCE(valid_4min, 0),
		         final_status, score, timestamp, COALESCE(title, '')
		         FROM signals ORDER BY id DESC LIMIT %d`, limit)
		rows, err = db.Query(query)
	} else {
		query = fmt.Sprintf(`SELECT id, symbol, signal_type, entry_price, 
		         COALESCE(price_1min, 0), COALESCE(price_2min, 0), COALESCE(price_4min, 0),
		         COALESCE(change_1min, 0), COALESCE(change_2min, 0), COALESCE(change_4min, 0),
		         COALESCE(valid_1min, 0), COALESCE(valid_2min, 0), COALESCE(valid_4min, 0),
		         final_status, score, timestamp, COALESCE(title, '')
		         FROM signals WHERE final_status = ? ORDER BY id DESC LIMIT %d`, limit)
		rows, err = db.Query(query, status)
	}

	if err != nil {
		return signals
	}
	defer rows.Close()

	for rows.Next() {
		var s Signal
		var v1, v2, v4 int
		rows.Scan(&s.ID, &s.Symbol, &s.SignalType, &s.EntryPrice,
			&s.Price1Min, &s.Price2Min, &s.Price4Min,
			&s.Change1Min, &s.Change2Min, &s.Change4Min,
			&v1, &v2, &v4,
			&s.FinalStatus, &s.Score, &s.Timestamp, &s.Title)
		s.Valid1Min = v1 != 0
		s.Valid2Min = v2 != 0
		s.Valid4Min = v4 != 0
		signals = append(signals, s)
	}
	return signals
}

func getSignalStats() SignalStats {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	var stats SignalStats

	db.QueryRow("SELECT COUNT(*) FROM signals WHERE final_status = 'valid'").Scan(&stats.Valid)
	db.QueryRow("SELECT COUNT(*) FROM signals WHERE final_status = 'invalid'").Scan(&stats.Invalid)
	db.QueryRow("SELECT COUNT(*) FROM signals WHERE final_status = 'pending'").Scan(&stats.Pending)

	total := stats.Valid + stats.Invalid
	if total > 0 {
		stats.Accuracy = float64(stats.Valid) / float64(total) * 100
	}

	return stats
}

func savePumpDump(pd PumpDump) {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	db.Exec(`INSERT INTO pump_dumps (id, symbol, price, prev_price, change_percent, 
	         event_type, volume, timestamp, saved_at) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)`,
		pd.ID, pd.Symbol, pd.Price, pd.PrevPrice, pd.ChangePercent,
		pd.EventType, pd.Volume, pd.Timestamp, time.Now().Format(time.RFC3339))
}

func getPumpDumps(limit int) []PumpDump {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	rows, _ := db.Query(`SELECT id, symbol, price, prev_price, change_percent, 
	                     event_type, volume, timestamp FROM pump_dumps 
	                     ORDER BY timestamp DESC LIMIT ?`, limit)
	defer rows.Close()

	var pds []PumpDump
	for rows.Next() {
		var pd PumpDump
		rows.Scan(&pd.ID, &pd.Symbol, &pd.Price, &pd.PrevPrice, &pd.ChangePercent,
			&pd.EventType, &pd.Volume, &pd.Timestamp)
		pds = append(pds, pd)
	}
	return pds
}

func getWhales(limit int) []Whale {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	rows, err := db.Query(`SELECT id, symbol, price, volume, change_percent, 
	                     whale_type, is_real, confidence_score, timestamp,
	                     COALESCE(title, ''), COALESCE(open_interest, 0), COALESCE(oi_delta, 0)
	                     FROM whales ORDER BY timestamp DESC LIMIT ?`, limit)
	if err != nil {
		return []Whale{}
	}
	defer rows.Close()

	var whales []Whale
	for rows.Next() {
		var w Whale
		rows.Scan(&w.ID, &w.Symbol, &w.Price, &w.Volume, &w.ChangePercent,
			&w.WhaleType, &w.IsReal, &w.ConfidenceScore, &w.Timestamp,
			&w.Title, &w.OpenInterest, &w.OIDelta)
		whales = append(whales, w)
	}
	return whales
}

func getWhaleFlow() WhaleFlow {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	var flow WhaleFlow

	since := time.Now().Add(-24 * time.Hour).Format(time.RFC3339)

	db.QueryRow(`SELECT COALESCE(SUM(volume), 0) FROM whales 
	             WHERE whale_type = 'buy' AND timestamp > ?`, since).Scan(&flow.Inflow)
	db.QueryRow(`SELECT COALESCE(SUM(volume), 0) FROM whales 
	             WHERE whale_type = 'sell' AND timestamp > ?`, since).Scan(&flow.Outflow)

	flow.Net = flow.Inflow - flow.Outflow

	return flow
}

func saveTrade(trade Trade) {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	db.Exec(`INSERT INTO trades (id, signal_id, symbol, side, entry_price, amount,
	         leverage, stop_loss, take_profit, exchange, status, opened_at, saved_at)
	         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'open', ?, ?)`,
		trade.ID, trade.SignalID, trade.Symbol, trade.Side, trade.EntryPrice,
		trade.Amount, trade.Leverage, trade.StopLoss, trade.TakeProfit,
		trade.Exchange, trade.OpenedAt, time.Now().Format(time.RFC3339))
}

func updateTrade(trade Trade) {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	db.Exec(`UPDATE trades SET exit_price = ?, pnl = ?, pnl_percent = ?, 
	         commission = ?, net_pnl = ?, status = 'closed', closed_at = ?,
	         close_reason = ?
	         WHERE id = ?`,
		trade.ExitPrice, trade.PnL, trade.PnLPercent, trade.Commission,
		trade.NetPnL, trade.ClosedAt, trade.CloseReason, trade.ID)
}

func getTrades(status string, limit int) []Trade {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	var query string
	var rows *sql.Rows

	if status == "" {
		query = `SELECT id, signal_id, symbol, side, entry_price, 
		         COALESCE(exit_price, 0), amount, leverage, 
		         COALESCE(pnl, 0), COALESCE(pnl_percent, 0), 
		         COALESCE(commission, 0), COALESCE(net_pnl, 0),
		         status, stop_loss, take_profit, exchange, opened_at, 
		         COALESCE(closed_at, ''), COALESCE(close_reason, '')
		         FROM trades ORDER BY opened_at DESC LIMIT ?`
		rows, _ = db.Query(query, limit)
	} else {
		query = `SELECT id, signal_id, symbol, side, entry_price, 
		         COALESCE(exit_price, 0), amount, leverage, 
		         COALESCE(pnl, 0), COALESCE(pnl_percent, 0), 
		         COALESCE(commission, 0), COALESCE(net_pnl, 0),
		         status, stop_loss, take_profit, exchange, opened_at, 
		         COALESCE(closed_at, ''), COALESCE(close_reason, '')
		         FROM trades WHERE status = ? 
		         ORDER BY opened_at DESC LIMIT ?`
		rows, _ = db.Query(query, status, limit)
	}
	defer rows.Close()

	var trades []Trade
	for rows.Next() {
		var t Trade
		rows.Scan(&t.ID, &t.SignalID, &t.Symbol, &t.Side, &t.EntryPrice,
			&t.ExitPrice, &t.Amount, &t.Leverage, &t.PnL, &t.PnLPercent,
			&t.Commission, &t.NetPnL, &t.Status, &t.StopLoss, &t.TakeProfit,
			&t.Exchange, &t.OpenedAt, &t.ClosedAt, &t.CloseReason)
		trades = append(trades, t)
	}
	return trades
}

func getTradeStats(period string) TradeStats {
	dbMutex.Lock()
	defer dbMutex.Unlock()

	var since string
	if period == "daily" {
		since = time.Now().Format("2006-01-02")
	} else {
		since = time.Now().Format("2006-01") + "-01"
	}

	var stats TradeStats

	row := db.QueryRow(`
		SELECT COUNT(*), 
		       COALESCE(SUM(CASE WHEN net_pnl > 0 THEN 1 ELSE 0 END), 0),
		       COALESCE(SUM(CASE WHEN net_pnl < 0 THEN 1 ELSE 0 END), 0),
		       COALESCE(SUM(net_pnl), 0),
		       COALESCE(SUM(commission), 0)
		FROM trades WHERE status = 'closed' AND DATE(opened_at) >= ?`, since)

	row.Scan(&stats.TotalTrades, &stats.Wins, &stats.Losses,
		&stats.TotalPnL, &stats.TotalCommission)

	if stats.TotalTrades > 0 {
		stats.WinRate = float64(stats.Wins) / float64(stats.TotalTrades) * 100
	}

	return stats
}

// ═══════════════════════════════════════════════════════════════
// API صرافی
// ═══════════════════════════════════════════════════════════════

func signBybit(params map[string]string, secretKey string) string {
	keys := make([]string, 0, len(params))
	for k := range params {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	var parts []string
	for _, k := range keys {
		parts = append(parts, k+"="+params[k])
	}
	queryString := strings.Join(parts, "&")

	h := hmac.New(sha256.New, []byte(secretKey))
	h.Write([]byte(queryString))
	return hex.EncodeToString(h.Sum(nil))
}

func getAccountInfo() AccountInfo {
	if config.APIKey == "" || config.SecretKey == "" {
		if config.PaperTrade {
			return AccountInfo{
				Success:   true,
				Exchange:  strings.ToUpper(config.Exchange),
				Name:      "Paper",
				UID:       "PAPER",
				Balance:   1000.0,
				Available: 1000.0,
				Locked:    0,
			}
		}
		return AccountInfo{Success: false, Error: "API Keys not set"}
	}
	if strings.ToLower(config.Exchange) == "bybit" {
		return getAccountBybit()
	}
	return getAccountOKX()
}

func getAccountBybit() AccountInfo {
	ts := fmt.Sprintf("%d", time.Now().UnixMilli())
	params := map[string]string{
		"accountType": "UNIFIED",
		"timestamp":   ts,
		"apiKey":      config.APIKey,
		"recvWindow":  "5000",
	}
	sign := signBybit(params, config.SecretKey)
	req, _ := http.NewRequest("GET", "https://api.bybit.com/v5/account/wallet-balance?accountType=UNIFIED", nil)
	req.Header.Set("X-BAPI-API-KEY", config.APIKey)
	req.Header.Set("X-BAPI-TIMESTAMP", ts)
	req.Header.Set("X-BAPI-RECV-WINDOW", "5000")
	req.Header.Set("X-BAPI-SIGN", sign)
	client := getHTTPClient()
	resp, err := client.Do(req)
	if err != nil {
		return AccountInfo{Success: false, Error: err.Error()}
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)
	var r struct {
		RetCode int    `json:"retCode"`
		RetMsg  string `json:"retMsg"`
		Result  struct {
			List []struct {
				TotalEquity         string `json:"totalEquity"`
				AvailableToWithdraw string `json:"availableToWithdraw"`
				AccountType         string `json:"accountType"`
			} `json:"list"`
		} `json:"result"`
	}
	if json.Unmarshal(body, &r) != nil || r.RetCode != 0 || len(r.Result.List) == 0 {
		return AccountInfo{Success: false, Error: r.RetMsg}
	}
	bal, _ := strconv.ParseFloat(r.Result.List[0].TotalEquity, 64)
	avail, _ := strconv.ParseFloat(r.Result.List[0].AvailableToWithdraw, 64)
	return AccountInfo{
		Success:   true,
		Exchange:  "BYBIT",
		Name:      "Unified",
		UID:       "",
		Balance:   bal,
		Available: avail,
		Locked:    math.Max(bal-avail, 0),
	}
}

func okxSign(ts, method, path string, body string, secret string) string {
	msg := ts + method + path + body
	h := hmac.New(sha256.New, []byte(secret))
	h.Write([]byte(msg))
	return base64.StdEncoding.EncodeToString(h.Sum(nil))
}

func getAccountOKX() AccountInfo {
	ts := time.Now().UTC().Format(time.RFC3339Nano)
	path := "/api/v5/account/balance"
	sign := okxSign(ts, "GET", path, "", config.SecretKey)
	req, _ := http.NewRequest("GET", "https://www.okx.com"+path, nil)
	req.Header.Set("OK-ACCESS-KEY", config.APIKey)
	req.Header.Set("OK-ACCESS-PASSPHRASE", config.OKXPass)
	req.Header.Set("OK-ACCESS-SIGN", sign)
	req.Header.Set("OK-ACCESS-TIMESTAMP", ts)
	client := getHTTPClient()
	resp, err := client.Do(req)
	if err != nil {
		return AccountInfo{Success: false, Error: err.Error()}
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)
	var r struct {
		Code string `json:"code"`
		Msg  string `json:"msg"`
		Data []struct {
			TotalEq string `json:"totalEq"`
		} `json:"data"`
	}
	if json.Unmarshal(body, &r) != nil || r.Code != "0" || len(r.Data) == 0 {
		return AccountInfo{Success: false, Error: r.Msg}
	}
	bal, _ := strconv.ParseFloat(r.Data[0].TotalEq, 64)
	return AccountInfo{
		Success:   true,
		Exchange:  "OKX",
		Name:      "Account",
		UID:       "",
		Balance:   bal,
		Available: bal,
		Locked:    0,
	}
}

func placeOrder(trade Trade) {
	if config.PaperTrade {
		log.Printf("� PaperTrade: %s %s $%.2f", trade.Symbol, trade.Side, trade.Amount)
		return
	}
	ex := strings.ToLower(config.Exchange)
	if ex == "bybit" {
		placeOrderBybit(trade)
	} else {
		placeOrderOKX(trade)
	}
}

func placeOrderBybit(trade Trade) {
	ts := fmt.Sprintf("%d", time.Now().UnixMilli())
	side := "Buy"
	if trade.Side == "SHORT" {
		side = "Sell"
	}
	qty := fmt.Sprintf("%.6f", trade.Amount/trade.EntryPrice)
	params := map[string]string{
		"category":   "linear",
		"symbol":     trade.Symbol,
		"side":       side,
		"orderType":  "Market",
		"qty":        qty,
		"timestamp":  ts,
		"apiKey":     config.APIKey,
		"recvWindow": "5000",
	}
	sign := signBybit(params, config.SecretKey)
	body := map[string]string{
		"category":  "linear",
		"symbol":    trade.Symbol,
		"side":      side,
		"orderType": "Market",
		"qty":       qty,
	}
	b, _ := json.Marshal(body)
	req, _ := http.NewRequest("POST", "https://api.bybit.com/v5/order/create", bytes.NewReader(b))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-BAPI-API-KEY", config.APIKey)
	req.Header.Set("X-BAPI-TIMESTAMP", ts)
	req.Header.Set("X-BAPI-RECV-WINDOW", "5000")
	req.Header.Set("X-BAPI-SIGN", sign)
	client := getHTTPClient()
	resp, err := client.Do(req)
	if err != nil {
		log.Printf("Bybit order error: %v", err)
		return
	}
	defer resp.Body.Close()
	io.ReadAll(resp.Body)
}

func placeOrderOKX(trade Trade) {
	inst := strings.Replace(trade.Symbol, "USDT", "-USDT-SWAP", 1)
	side := "buy"
	if trade.Side != "LONG" {
		side = "sell"
	}
	sz := fmt.Sprintf("%.6f", trade.Amount/trade.EntryPrice)
	body := map[string]string{
		"instId":  inst,
		"tdMode":  "cross",
		"side":    side,
		"ordType": "market",
		"sz":      sz,
	}
	b, _ := json.Marshal(body)
	ts := time.Now().UTC().Format(time.RFC3339Nano)
	path := "/api/v5/trade/order"
	sign := okxSign(ts, "POST", path, string(b), config.SecretKey)
	req, _ := http.NewRequest("POST", "https://www.okx.com"+path, bytes.NewReader(b))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("OK-ACCESS-KEY", config.APIKey)
	req.Header.Set("OK-ACCESS-PASSPHRASE", config.OKXPass)
	req.Header.Set("OK-ACCESS-SIGN", sign)
	req.Header.Set("OK-ACCESS-TIMESTAMP", ts)
	client := getHTTPClient()
	resp, err := client.Do(req)
	if err != nil {
		log.Printf("OKX order error: %v", err)
		return
	}
	defer resp.Body.Close()
	io.ReadAll(resp.Body)
}

// ═══════════════════════════════════════════════════════════════
// HTTP Handlers
// ═══════════════════════════════════════════════════════════════

func handleConfig(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	if r.Method == "POST" {
		json.NewDecoder(r.Body).Decode(&config)
		json.NewEncoder(w).Encode(map[string]interface{}{"success": true, "config": config})
		return
	}

	json.NewEncoder(w).Encode(config)
}

func handleMarket(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	source := r.URL.Query().Get("source")
	if source == "" {
		source = config.APISource
	}

	data, srcUsed, err := fetchMarketDataBest(source)
	if err != nil {
		json.NewEncoder(w).Encode(map[string]interface{}{
			"success": false,
			"error":   err.Error(),
			"source":  srcUsed,
		})
		return
	}

	whales := []Whale{}
	pumpDumps := []PumpDump{}
	if srcUsed != "demo" {
		whales = detectWhales(data)
		pumpDumps = detectPumpDumps(data)
		checkPendingSignals(data)
	}

	json.NewEncoder(w).Encode(map[string]interface{}{
		"success":    true,
		"data":       data,
		"source":     srcUsed,
		"whales":     whales,
		"pump_dumps": pumpDumps,
	})
}

func handleCurrentPrice(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	symbol := r.URL.Query().Get("symbol")
	if symbol == "" {
		json.NewEncoder(w).Encode(map[string]interface{}{"success": false, "error": "Symbol required"})
		return
	}

	data, err := fetchMarketDataCached(config.APISource)
	if err != nil {
		json.NewEncoder(w).Encode(map[string]interface{}{"success": false, "error": err.Error()})
		return
	}

	for _, m := range data {
		if m.Symbol == symbol {
			json.NewEncoder(w).Encode(map[string]interface{}{
				"success": true,
				"symbol":  m.Symbol,
				"price":   m.Price,
				"change":  m.Change,
				"volume":  m.Volume,
			})
			return
		}
	}

	json.NewEncoder(w).Encode(map[string]interface{}{"success": false, "error": "Symbol not found"})
}

func handleSignals(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	status := r.URL.Query().Get("status")
	signals := getSignals(status, 100)
	stats := getSignalStats()

	json.NewEncoder(w).Encode(map[string]interface{}{
		"signals": signals,
		"stats":   stats,
	})
}

func handleWhales(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	whales := getWhales(100)
	flow := getWhaleFlow()

	json.NewEncoder(w).Encode(map[string]interface{}{
		"whales": whales,
		"flow":   flow,
	})
}

func handlePumpDumps(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	json.NewEncoder(w).Encode(getPumpDumps(50))
}

func handleCandles(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")
	symbol := r.URL.Query().Get("symbol")
	source := r.URL.Query().Get("source")
	if source == "" {
		source = config.APISource
	}
	if symbol == "" {
		symbol = "BTCUSDT"
	}
	var candles []Candle
	var err error
	if source == "okx" {
		candles, err = fetchOKXCandles(symbol, "1m", 200)
	} else {
		candles, err = fetchBybitCandles(symbol, "1", 200)
	}
	if err != nil {
		json.NewEncoder(w).Encode(map[string]interface{}{"success": false})
		return
	}
	json.NewEncoder(w).Encode(map[string]interface{}{"success": true, "candles": candles})
}

func handleUTBot(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")
	symbol := r.URL.Query().Get("symbol")
	source := r.URL.Query().Get("source")
	if source == "" {
		source = config.APISource
	}
	if symbol == "" {
		symbol = "BTCUSDT"
	}
	data, err := buildUTBot(symbol, source)
	if err != nil {
		json.NewEncoder(w).Encode(map[string]interface{}{"success": false})
		return
	}
	json.NewEncoder(w).Encode(map[string]interface{}{"success": true, "data": data})
}

func handleHealth(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")
	bybitOK := true
	okxOK := true
	bitgetOK := true
	krakenOK := true
	bybitErr := ""
	okxErr := ""
	bitgetErr := ""
	krakenErr := ""
	if _, err := fetchBybitFutures(); err != nil {
		bybitOK = false
		bybitErr = err.Error()
	}
	if _, err := fetchOKXFutures(); err != nil {
		okxOK = false
		okxErr = err.Error()
	}
	if _, err := fetchBitgetFutures(); err != nil {
		bitgetOK = false
		bitgetErr = err.Error()
	}
	if _, err := fetchKrakenFutures(); err != nil {
		krakenOK = false
		krakenErr = err.Error()
	}
	json.NewEncoder(w).Encode(map[string]interface{}{
		"bybit_ok":   bybitOK,
		"okx_ok":     okxOK,
		"bybit_err":  bybitErr,
		"okx_err":    okxErr,
		"bitget_ok":  bitgetOK,
		"kraken_ok":  krakenOK,
		"bitget_err": bitgetErr,
		"kraken_err": krakenErr,
	})
}
func handleAccount(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	json.NewEncoder(w).Encode(getAccountInfo())
}

func handleTradeQueue(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	json.NewEncoder(w).Encode(getValidSignalsForTrade(config.MinScoreForTrade))
}

func handleAutoTradeStart(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	autoTrader.Start()
	json.NewEncoder(w).Encode(map[string]interface{}{"success": true, "message": "اتو ترید شروع شد"})
}

func handleAutoTradeStop(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	autoTrader.Stop()
	json.NewEncoder(w).Encode(map[string]interface{}{"success": true, "message": "اتو ترید متوقف شد"})
}

func handleAutoTradeStats(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	json.NewEncoder(w).Encode(map[string]interface{}{
		"is_running":         autoTrader.IsRunning,
		"daily_trades":       autoTrader.DailyTrades,
		"consecutive_losses": autoTrader.ConsecutiveLosses,
		"pnl":                autoTrader.PnL,
		"commission":         autoTrader.TotalCommission,
		"net_pnl":            autoTrader.PnL - autoTrader.TotalCommission,
		"open_trades":        len(autoTrader.OpenTrades),
	})
}

func handleTrades(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	status := r.URL.Query().Get("status")
	trades := getTrades(status, 100)
	dailyStats := getTradeStats("daily")
	monthlyStats := getTradeStats("monthly")

	json.NewEncoder(w).Encode(map[string]interface{}{
		"trades":        trades,
		"daily_stats":   dailyStats,
		"monthly_stats": monthlyStats,
	})
}

func handleIndex(w http.ResponseWriter, r *http.Request) {
	http.ServeFile(w, r, "./index_improved.html")
}

func handleMarketStream(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/event-stream")
	w.Header().Set("Cache-Control", "no-cache")
	w.Header().Set("Connection", "keep-alive")
	source := r.URL.Query().Get("source")
	if source == "" {
		source = config.APISource
	}
	flusher, ok := w.(http.Flusher)
	if !ok {
		return
	}
	for i := 0; i < 300; i++ {
		data, srcUsed, err := fetchMarketDataBest(source)
		var payload []byte
		if err == nil {
			b, _ := json.Marshal(map[string]interface{}{"success": true, "source": srcUsed, "data": data})
			payload = b
		} else {
			b, _ := json.Marshal(map[string]interface{}{"success": false, "error": err.Error()})
			payload = b
		}
		fmt.Fprintf(w, "data: %s\n\n", string(payload))
		flusher.Flush()
		time.Sleep(3 * time.Second)
	}
}

var wsUpgrader = websocket.Upgrader{
	CheckOrigin: func(r *http.Request) bool { return true },
}

var wsMarketCache = struct {
	sync.RWMutex
	data map[string]MarketData
}{data: make(map[string]MarketData)}
var bybitWSRunning bool

func bybitWSSubscribeAll() []string {
	list, _ := fetchBybitFutures()
	args := []string{}
	count := 0
	for _, m := range list {
		if strings.HasSuffix(m.Symbol, "USDT") {
			args = append(args, "tickers."+m.Symbol)
			count++
			if count >= 80 {
				break
			}
		}
	}
	if len(args) == 0 {
		args = []string{"tickers.BTCUSDT", "tickers.ETHUSDT"}
	}
	return args
}

func getWSNetDialer() func(network, addr string) (net.Conn, error) {
	if config.ProxyURL == "" {
		return nil
	}
	if strings.HasPrefix(config.ProxyURL, "socks5://") || strings.HasPrefix(config.ProxyURL, "socks5h://") {
		u := strings.TrimPrefix(strings.TrimPrefix(config.ProxyURL, "socks5://"), "socks5h://")
		d, err := proxy.SOCKS5("tcp", u, nil, proxy.Direct)
		if err == nil {
			return func(network, addr string) (net.Conn, error) { return d.Dial(network, addr) }
		}
	}
	return nil
}

func ensureBybitWS() {
	if bybitWSRunning {
		return
	}
	bybitWSRunning = true
	go func() {
		defer func() { bybitWSRunning = false }()
		for {
			dialer := &websocket.Dialer{
				HandshakeTimeout:  10 * time.Second,
				EnableCompression: true,
			}
			if nd := getWSNetDialer(); nd != nil {
				dialer.NetDial = nd
			}
			wsURL := "wss://stream.bybit.com/v5/public/linear"
			conn, _, err := dialer.Dial(wsURL, nil)
			if err != nil {
				time.Sleep(3 * time.Second)
				continue
			}
			sub := map[string]interface{}{"op": "subscribe", "args": bybitWSSubscribeAll()}
			_ = conn.WriteJSON(sub)
			for {
				_, message, err := conn.ReadMessage()
				if err != nil {
					_ = conn.Close()
					break
				}
				var pkt struct {
					Topic string `json:"topic"`
					Type  string `json:"type"`
					Data  []struct {
						Symbol       string `json:"symbol"`
						LastPrice    string `json:"lastPrice"`
						Price24hPcnt string `json:"price24hPcnt"`
						HighPrice24h string `json:"highPrice24h"`
						LowPrice24h  string `json:"lowPrice24h"`
						Turnover24h  string `json:"turnover24h"`
						OpenInterest string `json:"openInterest"`
						FundingRate  string `json:"fundingRate"`
						Timestamp    int64  `json:"timestamp"`
					} `json:"data"`
				}
				if json.Unmarshal(message, &pkt) == nil && strings.HasPrefix(pkt.Topic, "tickers.") {
					wsMarketCache.Lock()
					ts := time.Now().Format(time.RFC3339)
					for _, d := range pkt.Data {
						price, _ := strconv.ParseFloat(d.LastPrice, 64)
						changePct, _ := strconv.ParseFloat(d.Price24hPcnt, 64)
						high, _ := strconv.ParseFloat(d.HighPrice24h, 64)
						low, _ := strconv.ParseFloat(d.LowPrice24h, 64)
						volume, _ := strconv.ParseFloat(d.Turnover24h, 64)
						oi, _ := strconv.ParseFloat(d.OpenInterest, 64)
						funding, _ := strconv.ParseFloat(d.FundingRate, 64)
						prev := previousOI[d.Symbol]
						oiDelta := 0.0
						if prev > 0 {
							oiDelta = ((oi - prev) / prev) * 100
						}
						previousOI[d.Symbol] = oi
						wsMarketCache.data[d.Symbol] = MarketData{
							Symbol:       d.Symbol,
							Price:        price,
							Change:       changePct * 100,
							High:         high,
							Low:          low,
							Volume:       volume,
							Timestamp:    ts,
							Source:       "bybit_ws",
							IsFutures:    true,
							OpenInterest: oi,
							FundingRate:  funding * 100,
							OIDelta:      oiDelta,
						}
					}
					wsMarketCache.Unlock()
				}
			}
			time.Sleep(2 * time.Second)
		}
	}()
}

func handleMarketWS(w http.ResponseWriter, r *http.Request) {
	conn, err := wsUpgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}
	defer conn.Close()
	source := r.URL.Query().Get("source")
	if source == "" {
		source = config.APISource
	}
	for i := 0; i < 1200; i++ {
		var payload []byte
		if config.UseExchangeWS && strings.ToLower(source) == "bybit" {
			ensureBybitWS()
			wsMarketCache.RLock()
			snapshot := make([]MarketData, 0, len(wsMarketCache.data))
			for _, m := range wsMarketCache.data {
				snapshot = append(snapshot, m)
			}
			wsMarketCache.RUnlock()
			if len(snapshot) > 0 {
				// ایجاد سیگنال و اعتبارسنجی از داده‌های WS
				whales := detectWhales(snapshot)
				_ = whales
				detectPumpDumps(snapshot)
				checkPendingSignals(snapshot)
				b, _ := json.Marshal(map[string]interface{}{"success": true, "source": "bybit_ws", "data": snapshot})
				payload = b
			}
		}
		if payload == nil {
			data, srcUsed, err := fetchMarketDataBest(source)
			if err == nil {
				if srcUsed != "demo" {
					whales := detectWhales(data)
					_ = whales
					detectPumpDumps(data)
					checkPendingSignals(data)
				}
				b, _ := json.Marshal(map[string]interface{}{"success": true, "source": srcUsed, "data": data})
				payload = b
			} else {
				b, _ := json.Marshal(map[string]interface{}{"success": false, "error": err.Error()})
				payload = b
			}
		}
		if conn.WriteMessage(websocket.TextMessage, payload) != nil {
			break
		}
		time.Sleep(2 * time.Second)
	}
}

// ═══════════════════════════════════════════════════════════════
// Main
// ═══════════════════════════════════════════════════════════════

func main() {
	fmt.Println("══════════════════════════════════════════════════════════════")
	fmt.Println("   🐋 Whale Hunter Pro v6.0 - Enhanced Edition")
	fmt.Println("══════════════════════════════════════════════════════════════")
	fmt.Println()
	fmt.Println("   ✨ بهبودهای نسخه 6.0:")
	fmt.Println("   • Data Grid مدرن با فیلتر، سورت و جستجو")
	fmt.Println("   • اعتبارسنجی دقیق با پارامترهای پویا")
	fmt.Println("   • پشتیبانی Bybit و OKX فیوچرز")
	fmt.Println("   • نمایش قیمت فعلی با راست کلیک")
	fmt.Println("   • صفحه چارت UT_Bot Alert")
	fmt.Println("   • کنترل موقعیت‌های منقضی")
	fmt.Println("   • UI/UX بهبود یافته")
	fmt.Println()
	fmt.Println("   🌐 آدرس: http://localhost:8081")
	fmt.Println("══════════════════════════════════════════════════════════════")

	initDB()
	migrateDB()
	initBrainDB()
	initRedis()

	http.HandleFunc("/", handleIndex)
	http.HandleFunc("/api/config", handleConfig)
	http.HandleFunc("/api/market", handleMarket)
	http.HandleFunc("/api/current-price", handleCurrentPrice)
	http.HandleFunc("/api/signals", handleSignals)
	http.HandleFunc("/api/whales", handleWhales)
	http.HandleFunc("/api/pump-dumps", handlePumpDumps)
	http.HandleFunc("/api/candles", handleCandles)
	http.HandleFunc("/api/utbot", handleUTBot)
	http.HandleFunc("/api/health", handleHealth)
	http.HandleFunc("/api/account", handleAccount)
	http.HandleFunc("/api/trade-queue", handleTradeQueue)
	http.HandleFunc("/api/auto-trade/start", handleAutoTradeStart)
	http.HandleFunc("/api/auto-trade/stop", handleAutoTradeStop)
	http.HandleFunc("/api/auto-trade/stats", handleAutoTradeStats)
	http.HandleFunc("/api/trades", handleTrades)
	http.HandleFunc("/api/brain/learn-zone", handleBrainLearnZone)
	http.HandleFunc("/api/stream/market", handleMarketStream)
	http.HandleFunc("/ws/market", handleMarketWS)

	addr := fmt.Sprintf(":%d", config.Port)
	log.Printf("🚀 سرور در حال اجرا روی http://localhost:%d\n", config.Port)
	log.Fatal(http.ListenAndServe(addr, nil))
}
