# Hướng Dẫn Tích Hợp Tối Ưu Hóa Hình Ảnh Trong Laravel
## 1. Cài Đặt Các Package Cần Thiết
```bash
# Cài đặt Laravel Image Optimizer
composer require spatie/laravel-image-optimizer

# Cài đặt Intervention Image để resize ảnh
composer require intervention/image
```
## 2. Xuất File Config (Tùy Chọn)
```bash
php artisan vendor:publish --provider="Spatie\LaravelImageOptimizer\ImageOptimizerServiceProvider"
```
Tạo Symbolic Link cho Storage (nếu chưa có)
```bash
php artisan storage:link
```
## 3. Tạo Helper Class OptimizedImage
Tạo file app/Helpers/OptimizedImage.php:
```bash
<?php

namespace App\Helpers;

use Illuminate\Support\Facades\File;

class OptimizedImage
{
    /**
     * Lấy URL của phiên bản tối ưu hóa
     * 
     * @param string $originalUrl URL gốc của hình ảnh
     * @param string $size Kích thước (thumbnail, small, medium, large)
     * @return string URL của phiên bản đã tối ưu
     */
    public static function get($originalUrl, $size = 'medium')
    {
        if (empty($originalUrl)) {
            return asset('images/no-image.jpg');
        }
        
        // Lấy đường dẫn tương đối từ URL
        $relativePath = self::getRelativePathFromUrl($originalUrl);
        
        if (empty($relativePath)) {
            return $originalUrl;
        }
        
        // Đường dẫn đến phiên bản tối ưu
        $optimizedPath = "storage/optimized/{$size}/{$relativePath}";
        $fullOptimizedPath = public_path($optimizedPath);
        
        // Kiểm tra xem phiên bản tối ưu có tồn tại không
        if (File::exists($fullOptimizedPath)) {
            return asset($optimizedPath);
        }
        
        // Nếu không có phiên bản tối ưu, trả về URL gốc
        return $originalUrl;
    }
    /**
     * Tạo danh sách srcset cho hình ảnh
     *
     * @param string $originalUrl URL gốc của hình ảnh
     * @param array $sizes Các kích thước cần tạo srcset
     * @return string Danh sách srcset
     */
    public static function srcset($originalUrl, $sizes = ['thumbnail', 'small', 'medium', 'large'])
    {
        if (empty($originalUrl)) {
            return '';
        }
        
        $srcsetItems = [];
        $widthMap = [
            'thumbnail' => '150w',
            'small' => '300w',
            'medium' => '600w',
            'large' => '1200w'
        ];
        
        foreach ($sizes as $size) {
            $url = self::get($originalUrl, $size);
            if (isset($widthMap[$size])) {
                $srcsetItems[] = "{$url} {$widthMap[$size]}";
            }
        }
        
        return implode(', ', $srcsetItems);
    }
    
    /**
     * Lấy đường dẫn tương đối từ URL
     *
     * @param string $url URL đầy đủ hoặc tương đối
     * @return string Đường dẫn tương đối
     */
    private static function getRelativePathFromUrl($url)
    {
        if (empty($url)) return '';
        
        // Xử lý URL tuyệt đối
        if (strpos($url, 'http') === 0) {
            $parsedUrl = parse_url($url);
            $path = $parsedUrl['path'] ?? '';
            
            // Tìm phần '/storage/' trong đường dẫn
            $storagePos = strpos($path, '/storage/');
            if ($storagePos !== false) {
                return substr($path, $storagePos + 9); // +9 để bỏ qua '/storage/'
            }
            return '';
        } 
        // Xử lý URL tương đối
        elseif (strpos($url, '/storage/') === 0) {
            return substr($url, 9); // Bỏ qua '/storage/'
        }
        
        return $url;
    }
}
```
## 4. Tạo Command Tối Ưu Hóa Hình Ảnh
```bash
php artisan make:command OptimizeImages
```
Mở file app/Console/Commands/OptimizeImages.php và cập nhật nội dung:
```bash
<?php

namespace App\Console\Commands;

use Carbon\Carbon;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\File;
use Spatie\ImageOptimizer\OptimizerChainFactory;
use Intervention\Image\Facades\Image;

class OptimizeImages extends Command
{
    protected $signature = 'images:optimize {--force : Tối ưu lại tất cả hình ảnh} {--only-new : Chỉ tối ưu hình ảnh mới}';
    protected $description = 'Tối ưu hóa hình ảnh thông minh chỉ cho hình ảnh mới hoặc đã thay đổi';

    protected $sizes = [
        'thumbnail' => [150, 84],   // Tỷ lệ 16:9
        'small' => [238, 134],      // Kích thước trong section_poster_2.blade.php
        'medium' => [400, 225],     // Kích thước trung bình
        'large' => [485, 273]       // Kích thước trong section_poster_1.blade.php
    ];

    protected $optimizerChain;
    protected $lastOptimizedTime;
    protected $optimizedBasePath;
    protected $processedCount = 0;
    protected $skippedCount = 0;
    public function __construct()
    {
        parent::__construct();
        $this->optimizerChain = OptimizerChainFactory::create();
        $this->optimizedBasePath = storage_path('app/public/optimized');
    }

    public function handle()
    {
        $this->info('Đang kiểm tra cấu trúc thư mục...');
        $this->ensureDirectoriesExist();

        $forceAll = $this->option('force');
        $onlyNew = $this->option('only-new');

        $this->lastOptimizedTime = Cache::get('last_image_optimization', 0);
        $timeNow = time();

        if ($forceAll) {
            $this->info('Đang tối ưu lại TẤT CẢ hình ảnh...');
            $movies = DB::table('movies')->get();
        } elseif ($onlyNew) {
            $this->info('Đang tối ưu CHỈ những hình ảnh MỚI (chưa từng tối ưu)...');
            $movies = $this->getMoviesWithUnoptimizedImages();
        } else {
            $this->info('Đang tối ưu hình ảnh mới hoặc đã thay đổi từ lần cuối ('.Carbon::createFromTimestamp($this->lastOptimizedTime)->format('Y-m-d H:i:s').')...');
            $movies = DB::table('movies')
                ->where('created_at', '>=', date('Y-m-d H:i:s', $this->lastOptimizedTime))
                ->orWhere('updated_at', '>=', date('Y-m-d H:i:s', $this->lastOptimizedTime))
                ->get();
        }

        if ($movies->count() === 0) {
            $this->info('Không tìm thấy hình ảnh nào cần tối ưu.');
            return;
        }

        $this->info('Đã tìm thấy '.$movies->count().' phim có thể cần tối ưu hóa hình ảnh.');

        $bar = $this->output->createProgressBar($movies->count());
        $bar->start();

        foreach ($movies as $movie) {
            $posterUrl = $movie->poster_url ?? $movie->thumb_url ?? null;
            $thumbUrl = $movie->thumb_url ?? $movie->poster_url ?? null;
            
            if ($posterUrl) {
                $this->processImage($posterUrl);
            }
            
            if ($thumbUrl && $thumbUrl !== $posterUrl) {
                $this->processImage($thumbUrl);
            }
            
            $bar->advance();
        }

        $bar->finish();
        $this->newLine(2);

        // Cập nhật thời gian tối ưu cuối cùng
        Cache::put('last_image_optimization', $timeNow, now()->addYear());
        
        $this->info("Kết quả tối ưu hóa:");
        $this->info("- Đã xử lý: {$this->processedCount} hình ảnh");
        $this->info("- Đã bỏ qua: {$this->skippedCount} hình ảnh (không tìm thấy hoặc đã tối ưu)");
    }

    private function getMoviesWithUnoptimizedImages()
    {
        $allMovies = DB::table('movies')->get();
        $moviesNeedOptimization = collect();

        foreach ($allMovies as $movie) {
            $posterUrl = $movie->poster_url ?? $movie->thumb_url ?? null;
            $needsOptimization = false;

            if ($posterUrl && !$this->hasOptimizedVersions($posterUrl)) {
                $needsOptimization = true;
            }

            if ($needsOptimization) {
                $moviesNeedOptimization->push($movie);
            }
        }

        return $moviesNeedOptimization;
    }
    private function hasOptimizedVersions($imagePath)
    {
        $relativePath = $this->getRelativePathFromUrl($imagePath);
        if (empty($relativePath)) return false;

        // Kiểm tra phiên bản nhỏ - nếu có thì giả định các phiên bản khác cũng có
        $smallSizePath = $this->optimizedBasePath . '/small/' . $relativePath;
        return File::exists($smallSizePath);
    }

    private function processImage($imageUrl)
    {
        $relativePath = $this->getRelativePathFromUrl($imageUrl);
        
        if (empty($relativePath)) {
            $this->skippedCount++;
            return;
        }

        $publicPath = public_path('storage/' . $relativePath);
        
        if (!File::exists($publicPath)) {
            $this->skippedCount++;
            return;
        }

        $hasProcessed = false;
        
        foreach ($this->sizes as $size => $dimensions) {
            $targetDir = $this->optimizedBasePath . '/' . $size . '/' . dirname($relativePath);
            $targetPath = $targetDir . '/' . basename($relativePath);
            
            // Nếu không phải force và file đã tồn tại, bỏ qua
            if (!$this->option('force') && File::exists($targetPath)) {
                continue;
            }

            // Đảm bảo thư mục đích tồn tại
            if (!File::isDirectory($targetDir)) {
                File::makeDirectory($targetDir, 0755, true);
            }

            // Tạo phiên bản có kích thước phù hợp
            $resizedImage = Image::make($publicPath);
            
            // Resize thông minh giữ tỷ lệ
            $resizedImage->resize($dimensions[0], $dimensions[1], function ($constraint) {
                $constraint->aspectRatio();
                $constraint->upsize();
            });
            
            // Lưu phiên bản chưa tối ưu
            $resizedImage->save($targetPath, 90);
            
            // Tối ưu kích thước file
            $this->optimizerChain->optimize($targetPath);
            
            $hasProcessed = true;
        }

        if ($hasProcessed) {
            $this->processedCount++;
        } else {
            $this->skippedCount++;
        }
    }

    private function getRelativePathFromUrl($url)
    {
        if (empty($url)) return '';
        
        // Xử lý URL tương đối và tuyệt đối
        if (strpos($url, 'http') === 0) {
            $parsedUrl = parse_url($url);
            $path = $parsedUrl['path'] ?? '';
            
            // Tìm phần '/storage/' trong đường dẫn
            $storagePos = strpos($path, '/storage/');
            if ($storagePos !== false) {
                return substr($path, $storagePos + 9); // +9 để bỏ qua '/storage/'
            }
            return '';
        } else {
            // URL tương đối
            if (strpos($url, '/storage/') === 0) {
                return substr($url, 9); // Bỏ qua '/storage/'
            }
        }
        
        return $url;
    }

    private function ensureDirectoriesExist()
    {
        foreach ($this->sizes as $size => $dimensions) {
            $path = $this->optimizedBasePath . '/' . $size;
            if (!File::isDirectory($path)) {
                File::makeDirectory($path, 0755, true);
                $this->info("Đã tạo thư mục: $path");
            }
        }
    }
}
```
## 5. Tạo Thư Mục Cần Thiết
```bash
# Đảm bảo các thư mục tồn tại
mkdir -p storage/app/public/optimized/thumbnail
mkdir -p storage/app/public/optimized/small
mkdir -p storage/app/public/optimized/medium
mkdir -p storage/app/public/optimized/large

# Cấp quyền thích hợp
chmod -R 775 storage/app/public/optimized
```
## 6. Đăng Ký Command Trong Kernel
Mở file app/Console/Kernel.php và thêm command vào mảng $commands:
```bash
protected $commands = [
    // Các commands khác...
    \App\Console\Commands\OptimizeImages::class,
];
```
Thiết lập lịch trình tự động:
```bash
protected function schedule(Schedule $schedule)
{
    // Các lệnh schedule khác...
    
    // Chạy optimize cho hình ảnh mới mỗi giờ
    $schedule->command('images:optimize --only-new')->hourly();
    
    // Chạy tối ưu toàn bộ mỗi tuần vào thứ hai lúc 3 giờ sáng
    $schedule->command('images:optimize --force')->weekly()->mondays()->at('3:00');
}
```
## 7. Sử Dụng Helper Trong Blade Template
Cập nhật blade template để sử dụng helper:
```bash
<img 
    class="lazyload" 
    src="{{ asset('images/placeholder.webp') }}" 
    data-src="{{ \App\Helpers\OptimizedImage::get($movie->getPosterUrl(), 'medium') }}" 
    data-srcset="{{ \App\Helpers\OptimizedImage::srcset($movie->getPosterUrl()) }}" 
    sizes="(max-width: 768px) 300px, 600px" 
    alt="{{ $movie->getTitle() }}"
>
```
## 8. Chạy Command Lần Đầu Tiên
```bash
# Tối ưu tất cả hình ảnh lần đầu
php artisan images:optimize --force

# Hoặc chỉ tối ưu hình ảnh mới
php artisan images:optimize --only-new
```
