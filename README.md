# HttpClient.Caching
[![Version](https://img.shields.io/nuget/v/HttpClient.Caching.svg)](https://www.nuget.org/packages/HttpClient.Caching)  [![Downloads](https://img.shields.io/nuget/dt/HttpClient.Caching.svg)](https://www.nuget.org/packages/HttpClient.Caching)

<img src="https://raw.githubusercontent.com/thomasgalliker/HttpClient.Caching/master/logo.png" alt="HttpClient.Caching" align="right" width=140>
HttpClient.Caching adds http response caching to HttpClient.

### Download and Install HttpClient.Caching
This library is available on NuGet: https://www.nuget.org/packages/HttpClient.Caching/
Use the following command to install HttpClient.Caching using NuGet package manager console:

    PM> Install-Package HttpClient.Caching

You can use this library in any .Net project which is compatible to .Net Framework 4.5+ and .Net Standard 1.2+ (e.g. Xamarin Android, iOS, Universal Windows Platform, etc.)

### API Usage
#### Using MemoryCache
Declare IMemoryCache in your API service, either by creating an instance manually or by injecting IMemoryCache into your API service class.
```C#
private readonly IMemoryCache memoryCache = new MemoryCache();
```

Following example show how IMemoryCache can be used to store an HTTP GET result in memory for a given time span (cacheExpirection):
```C#
public async Task<TResult> GetAsync<TResult>(string uri, TimeSpan? cacheExpiration = null)
{
    var stopwatch = new Stopwatch();
    stopwatch.Start();

    TResult result;
    var caching = cacheExpiration.HasValue;
    if (caching && this.memoryCache.TryGetValue(uri, out result))
    {
        stopwatch.Stop();
        this.tracer.Debug($"{nameof(this.GetAsync)} for Uri '{uri}' finished in {stopwatch.Elapsed.ToSecondsString()} (caching=true)");
        return result;
    }

    var httpResponseMessage = await this.HandleRequest(() => this.httpClient.GetAsync(uri));
    var jsonResponse = await this.HandleResponse(httpResponseMessage);
    result = await Task.Run(() => JsonConvert.DeserializeObject<TResult>(jsonResponse, this.serializerSettings));

    if (caching)
    {
        this.memoryCache.Set(uri, result, cacheExpiration.Value);
    }
    else
    {
        this.memoryCache.Remove(uri);
    }

    stopwatch.Stop();
    this.tracer.Debug($"{nameof(this.GetAsync)} for Uri '{uri}' finished in {stopwatch.Elapsed.ToSecondsString()}");
    return result;
}
```

#### Using InMemoryCacheHandler
HttpClient allows to inject a custom http handler. In the follwing example, we inject an HttpClientHandler which is nested into an InMemoryCacheHandler where the InMemoryCacheHandler is responsible for maintaining and reading the cache.
```C#
static void Main(string[] args)
{
    const string url = "http://worldclockapi.com/api/json/utc/now";

    var httpClientHandler = new HttpClientHandler();
    var cacheExpirationPerHttpResponseCode = CacheExpirationProvider.CreateSimple(TimeSpan.FromSeconds(60), TimeSpan.FromSeconds(10), TimeSpan.FromSeconds(5));
    var handler = new InMemoryCacheHandler(httpClientHandler, cacheExpirationPerHttpResponseCode);
    using (var client = new HttpClient(handler))
    {
        // HttpClient calls the same API endpoint five times:
        // - The first attempt is called against the real API endpoint since no cache is available
        // - Attempts 2 to 5 can be read from cache
        for (var i = 1; i <= 5; i++)
        {
            Console.Write($"Attempt {i}: HTTP GET {url}...");
            var stopwatch = Stopwatch.StartNew();
            var result = client.GetAsync(url).GetAwaiter().GetResult();

            // Do something useful with the returned content...
            var content = result.Content.ReadAsStringAsync().GetAwaiter().GetResult();
            Console.WriteLine($" completed in {stopwatch.ElapsedMilliseconds}ms");

            // Artificial wait time...
            Thread.Sleep(1000);
        }
    }

    Console.WriteLine();

    StatsResult stats = handler.StatsProvider.GetStatistics();
    Console.WriteLine($"TotalRequests: {stats.Total.TotalRequests}");
    Console.WriteLine($"-> CacheHit: {stats.Total.CacheHit}");
    Console.WriteLine($"-> CacheMiss: {stats.Total.CacheMiss}");
    Console.ReadKey();
}
```

### License
This project is Copyright &copy; 2018 [Thomas Galliker](https://ch.linkedin.com/in/thomasgalliker). Free for non-commercial use. For commercial use please contact the author.
