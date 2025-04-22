# Brave MCP Sunucusu ve Claude Entegrasyonu

Bu döküman, Brave Search API'yi kullanan bir MCP (Model Context Protocol) sunucusunun nasıl çalıştığını ve Claude gibi AI modellerinin bu sunucuyla nasıl etkileşime girdiğini açıklar.

## Proje Hakkında

Bu proje, Brave Search API'sini kullanarak web aramaları ve yerel (local) iş yeri aramaları yapabilen bir arayüz oluşturuyor. Model Context Protocol (MCP) kullanılarak Claude gibi büyük dil modelleri için araç (tool) desteği sağlanıyor.

### Ana Bileşenler

1. **Web Arama Aracı**: "brave_web_search" adlı araç, genel sorgular, haberler ve çevrimiçi içerikler için web aramaları gerçekleştiriyor.

2. **Yerel Arama Aracı**: "brave_local_search" adlı araç, restoranlar, işletmeler ve hizmetler gibi fiziksel konumları aramak için kullanılıyor. Bu araç:
   - İşletme adı ve adreslerini
   - Derecelendirme ve yorum sayılarını
   - Telefon numaralarını ve çalışma saatlerini döndürüyor

3. **MCP Sunucusu**: Büyük dil modellerinin bu araçları kullanabilmesi için bir protokol sunucusu oluşturulmuş.

4. **Hız Sınırlaması**: API çağrılarının belirli limitleri aşmaması için bir hız sınırlama mekanizması eklenmiş.

## MCP Sunucusunun Çalışma Prensibi

### Araçların Tanımlanması

MCP sunucusu, araçları şu şekilde tanımlar:

```typescript
const WEB_SEARCH_TOOL: Tool = {
  name: "brave_web_search",
  description: "Performs a web search using the Brave Search API...",
  inputSchema: {
    type: "object",
    properties: {
      query: {
        type: "string",
        description: "Search query (max 400 chars, 50 words)"
      },
      count: {
        type: "number",
        description: "Number of results (1-20, default 10)",
        default: 10
      },
      offset: {
        type: "number",
        description: "Pagination offset (max 9, default 0)",
        default: 0
      },
    },
    required: ["query"],
  },
};
```

### İşleyicilerin (Handlers) Ayarlanması

MCP sunucusu iki temel işleyici kullanır:

1. **Araç Listeleme İşleyicisi**:

```typescript
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [WEB_SEARCH_TOOL, LOCAL_SEARCH_TOOL],
}));
```

Bu işleyici, AI modelinin (Claude gibi) kullanabileceği tüm araçları listeler.

2. **Araç Çağırma İşleyicisi**:

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  try {
    const { name, arguments: args } = request.params;

    if (!args) {
      throw new Error("No arguments provided");
    }

    switch (name) {
      case "brave_web_search": {
        // Web araması işlemleri
      }
      
      case "brave_local_search": {
        // Yerel arama işlemleri
      }
      
      default:
        // Bilinmeyen araç hatası
    }
  } catch (error) {
    // Hata yönetimi
  }
});
```

Bu işleyici, AI modelinin bir aracı çağırma isteğini işler.

## Claude'un MCP Araçlarını Kullanma Süreci

Claude'un araçları kullanma süreci şu adımları içerir:

1. **Araç İhtiyacının Tespit Edilmesi**:
   - Claude, kullanıcının mesajını analiz eder ve bu mesajın bir dış araç gerektirip gerektirmediğini değerlendirir.
   - Örneğin: "İstanbul'daki en iyi restoranlar nelerdir?" sorgusu güncel bilgi gerektirir.

2. **Uygun Aracın Seçilmesi**:
   - Claude önce `ListToolsRequestSchema` ile mevcut araçların listesini alır.
   - Soru tipine göre en uygun aracı seçer (web araması veya yerel arama).

3. **Araç Çağrısının Yapılması**:
   - Claude, araç tanımından parametre yapısını öğrenir (isimler, tipler, zorunlu alanlar).
   - Kullanıcının sorusunu analiz ederek uygun parametreleri belirler (ör. sorgu metni, sonuç sayısı).
   - `CallToolRequestSchema` ile MCP sunucusuna istek gönderir.

4. **Sonuçların İşlenmesi**:
   - MCP sunucusu aracı çalıştırıp sonuçları döndürür.
   - Claude, bu sonuçları kendi yanıtına entegre eder.

Önemli not: Kullanıcının doğrudan "web'de ara" demesi gerekmez; Claude sorunun doğasını anlayarak arka planda uygun aracı otomatik olarak seçer.

## Debug ve İzleme Teknikleri

MCP sunucusunu debug etmek veya izlemek için çeşitli yöntemler kullanılabilir:

### HTTP Debug Sunucusu

Bu yöntemle, MCP sunucusu çalışırken gerçekleşen işlemleri bir web tarayıcısı üzerinden izleyebilirsiniz:

```typescript
import http from 'http';

// Debug için log havuzu
const debugLogs: string[] = [];

// HTTP debug sunucusu
const debugServer = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/html' });
  res.end(`
    <html>
      <head>
        <meta http-equiv="refresh" content="5">
        <style>pre { background: #f0f0f0; padding: 10px; }</style>
      </head>
      <body>
        <h1>MCP Debug Logs</h1>
        <pre>${debugLogs.join('\n')}</pre>
      </body>
    </html>
  `);
});

// Log yazmak için kullanılacak fonksiyon
function logDebug(message: string) {
  const logEntry = `${new Date().toISOString()}: ${message}`;
  debugLogs.push(logEntry);
  if (debugLogs.length > 100) debugLogs.shift(); // Son 100 logu tut
}

// Debug sunucusunu başlat
debugServer.listen(3333, () => {
  console.error('Debug server running at http://localhost:3333');
});
```

Bu koddan sonra, izlemek istediğiniz yerlere `logDebug()` çağrıları ekleyebilirsiniz:

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  try {
    const { name, arguments: args } = request.params;
    
    // İstek geldiğinde log kaydı oluştur
    logDebug(`İstek alındı: ${name}, args: ${JSON.stringify(args)}`);
    
    // ... diğer kodlar ...
  } catch (error) {
    logDebug(`Hata: ${error instanceof Error ? error.message : String(error)}`);
    // ... hata işleme ...
  }
});
```

### Diğer Debug Yöntemleri

1. **Dosya Tabanlı Loglama**:
   - Önemli işlemleri bir log dosyasına yazarak izleyebilirsiniz.

2. **Environment Variable Kullanımı**:
   - Debug modunu çevre değişkenleriyle kontrol edebilirsiniz.

3. **Wrapper Script Oluşturma**:
   - Claude'un çalıştırdığı sunucu komutunu bir wrapper script ile sarıp stdin/stdout akışını izleyebilirsiniz.

## Sonuç

MCP protokolü, Claude gibi AI modellerinin dış dünya ile etkileşime geçmesini sağlayan güçlü bir yöntemdir. Bu proje sayesinde Claude, kullanıcıların sorularını yanıtlarken Brave Search API'sini kullanarak güncel ve doğru bilgilere erişebilir.

Araçları debug etmek, izlemek ve geliştirmek için yukarıda belirtilen yöntemler kullanılabilir, böylece AI model entegrasyonunuzun sorunsuz çalıştığından emin olabilirsiniz.
