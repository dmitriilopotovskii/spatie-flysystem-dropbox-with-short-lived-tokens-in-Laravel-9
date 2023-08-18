# Guide for use spatie/flysystem-dropbox with short-lived tokens in Laravel 9

##  spatie/flysystem-dropbox with short-lived tokens in Laravel 9|10

Install spatie/flysystem-dropbox
```
composer require spatie/flysystem-dropbox
```

add this settings in flysystems.php 
```
  'disks' => [ 
  ...

        'dropbox' => [
        'driver' => 'dropbox',
        'app_key' => env('DROPBOX_APP_KEY'),
        'app_secret' => env('DROPBOX_APP_SECRET'),
        'refresh_token' => env('DROPBOX_REFRESH_TOKEN'),
],
```
[You should get from the dropbox api](https://developers.dropbox.com/oauth-guide)

    app_key

    app_secret

    refresh_token

Then add to the .env file with your values
```
   DROPBOX_APP_KEY=
   DROPBOX_APP_SECRET=
   DROPBOX_REFRESH_TOKEN=
```
And then create DropboxServiceProvider.php in app/providers

```php
<?php

namespace App\Providers;

use App\Services\AutoRefreshingDropBoxTokenService;
use Illuminate\Filesystem\FilesystemAdapter;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\ServiceProvider;
use League\Flysystem\Filesystem;
use Spatie\Dropbox\Client as DropboxClient;
use Spatie\FlysystemDropbox\DropboxAdapter;

class DropboxServiceProvider extends ServiceProvider
{
    /**
     * Register services.
     *
     * @return void
     */
    public function register()
    {
        //
    }

    /**
     * Bootstrap services.
     *
     * @return void
     */

    public function boot()
    {
        Storage::extend('dropbox', function () {
            $client = new DropboxClient(
                (new AutoRefreshingDropBoxTokenService())->getToken()
            );
            $adapter = new DropboxAdapter($client);

            return new FilesystemAdapter(
                new Filesystem($adapter, ['case_sensitive' => false]),
                $adapter
            );
        });
    }
}
```
Don`t forget register it in config/app.php
```php
 /*
         * Package Service Providers...
         */

App\Providers\DropboxServiceProvider::class,
```
Then create  AutoRefreshingDropBoxTokenService in App/Services

```php
<?php

namespace App\Services;

use Exception;
use GuzzleHttp\Client as HttpClient;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\ServiceProvider;
use Spatie\Dropbox\TokenProvider;

class AutoRefreshingDropBoxTokenService implements TokenProvider
{
    private string $key;
    private string $secret;
    private string $refreshToken;

    public function __construct()
    {
        $this->key = config('filesystems.disks.dropbox.app_key');
        $this->secret = config('filesystems.disks.dropbox.app_secret');
        $this->refreshToken = config('filesystems.disks.dropbox.refresh_token');
    }

    public function getToken(): string
    {
        return Cache::remember('access_token', 14000, function () {
            return $this->refreshToken();
        });
    }

    public function refreshToken(): string|bool
    {
        try {
            $client = new HttpClient();
            $res = $client->request(
                'POST',
                "https://{$this->key}:{$this->secret}@api.dropbox.com/oauth2/token",
                [
                    'form_params' => [
                        'grant_type' => 'refresh_token',
                        'refresh_token' => $this->refreshToken,
                    ],
                ]
            );
            if ($res->getStatusCode() == 200) {
                $response = json_decode($res->getBody(), true);
                return trim(json_encode($response['access_token']), '"');
            } else {
                return false;
            }
        } catch (Exception $e) {
            /*$this->logger->error("[{$e->getCode()}] {$e->getMessage()}");*/
            return false;
        }
    }
}

```
That`s all 

