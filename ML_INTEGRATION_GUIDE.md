# Panduan Integrasi Machine Learning untuk Perhitungan Bobot Pakan

## Lokasi File Machine Learning

File Machine Learning untuk perhitungan bobot pakan perlu ditempatkan di:

```
app/Services/ML/
```

### Struktur Direktori yang Disarankan:

```
app/
└── Services/
    └── ML/
        ├── FeedWeightPredictor.php      # Service class untuk ML prediction
        ├── models/                      # Folder untuk model ML (jika ada)
        │   └── feed_weight_model.pkl   # Contoh: model Python yang di-serialize
        └── scripts/                     # Folder untuk script ML (opsional)
            └── train_model.py          # Script untuk training model
```

## Endpoint API untuk Machine Learning

### 1. Endpoint untuk Mengambil Data Bobot

**URL:** `GET /weight/data`

**Response:**
```json
{
    "data": [
        {
            "cow_id": 1,
            "weight": 450.5,
            "measured_at": "2025-11-19"
        },
        ...
    ],
    "total_records": 50
}
```

### 2. Endpoint untuk Prediksi Bobot Pakan (Perlu Dibuat)

**URL:** `POST /weight/predict`

**Request Body:**
```json
{
    "cow_id": 1,
    "current_weight": 450.5,
    "target_weight": 460.0,
    "days": 30
}
```

**Response:**
```json
{
    "recommended_feed": 125.5,
    "unit": "kg",
    "confidence": 0.92,
    "breakdown": {
        "morning": 45.0,
        "afternoon": 40.5,
        "evening": 40.0
    }
}
```

## Cara Integrasi

### Opsi 1: Python ML Model via API

1. Buat service class di `app/Services/ML/FeedWeightPredictor.php`:

```php
<?php

namespace App\Services\ML;

use App\Models\CowWeight;
use Illuminate\Support\Facades\Http;

class FeedWeightPredictor
{
    private $mlApiUrl;

    public function __construct()
    {
        $this->mlApiUrl = config('services.ml.api_url', 'http://localhost:5000');
    }

    /**
     * Predict feed weight based on cow weight
     */
    public function predictFeedWeight(int $cowId, float $currentWeight, ?float $targetWeight = null, int $days = 30): array
    {
        // Get historical data
        $history = CowWeight::where('cow_id', $cowId)
            ->orderBy('measured_at', 'desc')
            ->limit(12)
            ->get();

        // Prepare data for ML model
        $data = [
            'cow_id' => $cowId,
            'current_weight' => $currentWeight,
            'target_weight' => $targetWeight,
            'days' => $days,
            'history' => $history->map(fn($w) => [
                'weight' => $w->weight,
                'date' => $w->measured_at->format('Y-m-d')
            ])->toArray()
        ];

        // Call ML API
        $response = Http::post("{$this->mlApiUrl}/predict", $data);

        if ($response->successful()) {
            return $response->json();
        }

        throw new \Exception('ML prediction failed: ' . $response->body());
    }
}
```

2. Tambahkan route di `routes/web.php`:

```php
Route::post('/weight/predict', [WeightController::class, 'predict'])->name('weight.predict');
```

3. Tambahkan method di `WeightController`:

```php
public function predict(Request $request, FeedWeightPredictor $predictor)
{
    $validated = $request->validate([
        'cow_id' => 'required|integer|min:1|max:10',
        'current_weight' => 'required|numeric|min:0',
        'target_weight' => 'nullable|numeric|min:0',
        'days' => 'required|integer|min:1|max:90',
    ]);

    try {
        $result = $predictor->predictFeedWeight(
            $validated['cow_id'],
            $validated['current_weight'],
            $validated['target_weight'] ?? null,
            $validated['days']
        );

        return response()->json($result);
    } catch (\Exception $e) {
        return response()->json([
            'error' => $e->getMessage()
        ], 500);
    }
}
```

### Opsi 2: Python ML Model via Command

1. Buat Artisan command:

```bash
php artisan make:command PredictFeedWeight
```

2. Di command, panggil script Python:

```php
public function handle()
{
    $cowId = $this->argument('cow_id');
    $weight = $this->argument('weight');
    
    $output = shell_exec("python app/Services/ML/scripts/predict.py {$cowId} {$weight}");
    
    $this->info($output);
}
```

### Opsi 3: Model PHP Native (untuk model sederhana)

Jika model ML cukup sederhana, bisa langsung diimplementasikan di PHP:

```php
namespace App\Services\ML;

class FeedWeightPredictor
{
    public function predictFeedWeight(float $currentWeight, float $targetWeight, int $days): float
    {
        // Simple linear regression example
        $weightDiff = $targetWeight - $currentWeight;
        $dailyGain = $weightDiff / $days;
        
        // Feed conversion ratio (kg feed per kg weight gain)
        $fcr = 3.5; // Example: 3.5 kg feed per 1 kg weight gain
        
        return $dailyGain * $fcr * $days;
    }
}
```

## Konfigurasi

Tambahkan di `.env`:

```env
ML_API_URL=http://localhost:5000
ML_ENABLED=true
ML_MODEL_PATH=app/Services/ML/models/feed_weight_model.pkl
```

Tambahkan di `config/services.php`:

```php
'ml' => [
    'api_url' => env('ML_API_URL', 'http://localhost:5000'),
    'enabled' => env('ML_ENABLED', false),
    'model_path' => env('ML_MODEL_PATH', 'app/Services/ML/models/feed_weight_model.pkl'),
],
```

## Data yang Tersedia untuk ML

Data bobot sapi dapat diakses melalui:

1. **Model Eloquent:** `App\Models\CowWeight`
2. **API Endpoint:** `GET /weight/data`
3. **Method Helper:** `CowWeight::getWeightsForChart()`

## Contoh Data Format

```json
{
    "cow_id": 1,
    "weight": 450.5,
    "measured_at": "2025-11-19",
    "notes": "Pengukuran bulanan"
}
```

## Langkah Selanjutnya

1. Buat model ML (Python/JavaScript/PHP)
2. Simpan model di `app/Services/ML/models/`
3. Buat service class untuk memanggil model
4. Integrasikan dengan `WeightController`
5. Tambahkan UI untuk menampilkan prediksi di halaman "Atur Bobot Pakan"

## Catatan Penting

- Pastikan data bobot sudah ada minimal 3-6 bulan untuk training model yang baik
- Model ML perlu di-update secara berkala dengan data baru
- Simpan hasil prediksi untuk evaluasi akurasi model
- Pertimbangkan faktor lain seperti umur sapi, jenis pakan, kondisi kesehatan

