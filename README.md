# Try-Catch Block in Laravel
> Guida pratica alla gestione delle eccezioni nel contesto Laravel

## Indice Rapido

- [Introduzione](#introduzione)
- [Cos'è il Try-Catch Block](#cosè-il-try-catch-block)
- [Cosa fa](#cosa-fa)
- [Best Practices](#best-practices)
- [Quando usarlo in Laravel](#quando-usarlo-in-laravel)
- [Quando non usarlo](#quando-non-usarlo)
- [Alternative Laravel](#alternative-laravel)
- [Esempi Pratici](#esempi-pratici)
- [Fonti e Riferimenti](#fonti-e-riferimenti)

---

## Introduzione

Il blocco `try-catch` è una struttura fondamentale in PHP per la gestione delle eccezioni. Nel contesto Laravel, l'uso del `try-catch` deve essere bilanciato: da un lato è necessario per gestire errori specifici, dall'altro Laravel offre meccanismi integrati che spesso rendono il `try-catch` esplicito non necessario o addirittura controproducente.

Questa guida ti aiuta a capire quando usare il `try-catch` e quando affidarti alle funzionalità native di Laravel.

---

## Cos'è il Try-Catch Block

Il blocco `try-catch` è una struttura di controllo che permette di:

1. **Eseguire codice potenzialmente problematico** all'interno del blocco `try`
2. **Intercettare eccezioni** generate durante l'esecuzione nel blocco `catch`
3. **Gestire gli errori** senza interrompere l'esecuzione dell'applicazione

**Struttura base:**

```php
try {
    // Codice che potrebbe generare un'eccezione
    $result = riskyOperation();
} catch (SpecificException $e) {
    // Gestione dell'eccezione specifica
    handleError($e);
} catch (\Exception $e) {
    // Gestione di altre eccezioni generiche
    handleGenericError($e);
} finally {
    // Codice eseguito sempre, anche in caso di eccezione
    cleanup();
}
```

---

## Cosa fa

Il blocco `try-catch` consente di:

- **Controllare il flusso di esecuzione** quando si verificano errori
- **Prevenire il crash dell'applicazione** gestendo errori previsti
- **Fornire feedback specifico** all'utente o al sistema
- **Eseguire operazioni di cleanup** attraverso il blocco `finally`
- **Loggare o reportare errori** in modo controllato

**Flusso di esecuzione:**

1. Il codice nel blocco `try` viene eseguito
2. Se viene lanciata un'eccezione, l'esecuzione si interrompe
3. Laravel cerca un blocco `catch` che corrisponda al tipo di eccezione
4. Se trovato, il codice nel `catch` viene eseguito
5. Il blocco `finally` (se presente) viene sempre eseguito

---

## Best Practices

### 1. Catturare eccezioni specifiche, non generiche

**❌ Evitare:** Catturare `\Exception` generico nasconde problemi critici.

**✅ Preferire:** Catturare eccezioni specifiche come `ModelNotFoundException`, `QueryException`, `ValidationException`.

### 2. Evitare try-catch nei controller (quando possibile)

**Regola generale:** Delegare la logica ai service e lasciare che l'Exception Handler globale gestisca le eccezioni. Usa Form Request per validazione e Policy per autorizzazione.

**❌ Evitare:** Gestire validazione, autorizzazione o errori generici direttamente nei controller con try-catch.

**✅ Preferire:** Usare Form Request per validazione e delegare la logica ai service. Lasciare che l'Exception Handler globale gestisca le eccezioni.

**⚠️ Quando invece conviene usarlo nei controller:**

Ci sono casi specifici in cui il try-catch nel controller è appropriato:

1. **Gestione di risposte HTTP specifiche per il contesto della richiesta**

```php
public function store(CreateOrderRequest $request, OrderService $service)
{
    try {
        $order = $service->createOrder($request->validated());
        return response()->json($order, 201);
    } catch (PaymentException $e) {
        // Risposta specifica per questo endpoint
        return response()->json([
            'error' => 'Payment failed',
            'message' => $e->getMessage(),
            'retry_url' => route('orders.retry-payment', $order->id)
        ], 422);
    }
}
```

2. **Operazioni che richiedono cleanup o azioni specifiche del controller**

```php
public function upload(Request $request)
{
    $file = $request->file('document');
    
    try {
        $path = $file->store('documents');
        $document = Document::create(['path' => $path]);
        
        return response()->json($document, 201);
    } catch (\Exception $e) {
        // Cleanup: elimina il file se la creazione del record fallisce
        if (isset($path) && Storage::exists($path)) {
            Storage::delete($path);
        }
        
        Log::error('Document upload failed', ['exception' => $e]);
        return response()->json(['error' => 'Upload failed'], 500);
    }
}
```

3. **Gestione di eccezioni che richiedono logica di routing o redirect specifica**

```php
public function processPayment($orderId)
{
    try {
        $order = Order::findOrFail($orderId);
        $result = app(PaymentService::class)->process($order);
        
        return redirect()->route('orders.show', $order->id)
            ->with('success', 'Payment processed');
    } catch (PaymentDeclinedException $e) {
        // Redirect specifico per pagamento rifiutato
        return redirect()->route('orders.payment', $orderId)
            ->with('error', 'Payment was declined. Please try again.');
    } catch (InsufficientFundsException $e) {
        // Redirect a pagina di ricarica
        return redirect()->route('account.top-up')
            ->with('error', 'Insufficient funds');
    }
}
```

**Principio guida:** Usa try-catch nel controller solo quando la gestione dell'eccezione richiede logica specifica del controller (risposte HTTP personalizzate, cleanup, redirect specifici). Altrimenti, lascia che l'Exception Handler globale gestisca tutto.

### 3. Usare l'Exception Handler globale

Configura la gestione centralizzata in `bootstrap/app.php` (Laravel 11+) o `app/Exceptions/Handler.php` (Laravel 10):

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->dontReport([
        ValidationException::class,
        AuthenticationException::class,
    ]);
    
    $exceptions->render(function (ModelNotFoundException $e, $request) {
        if ($request->expectsJson()) {
            return response()->json(['error' => 'Resource not found'], 404);
        }
    });
})
```

### 4. Creare eccezioni personalizzate per il dominio

Crea eccezioni specifiche per il tuo dominio con metodi `report()` e `render()` personalizzati.

### 5. Evitare try-catch vuoti

**❌ Evitare:** Catturare eccezioni senza gestirle o loggarle.

**✅ Preferire:** Sempre loggare o usare `report()` e fornire un fallback appropriato.

---

## Quando usarlo in Laravel

### 1. Operazioni critiche che richiedono gestione specifica

**Esempio: chiamate a servizi esterni**

```php
public function processPayment(PaymentData $data): PaymentResult
{
    try {
        $response = Http::timeout(10)
            ->post('https://payment-gateway.com/api/charge', $data->toArray());
        
        return PaymentResult::fromResponse($response);
    } catch (ConnectionException $e) {
        Log::warning('Payment gateway connection failed', ['exception' => $e]);
        throw new PaymentServiceUnavailableException('Service unavailable', 0, $e);
    } catch (TimeoutException $e) {
        Log::warning('Payment gateway timeout', ['exception' => $e]);
        throw new PaymentTimeoutException('Request timed out', 0, $e);
    }
}
```

### 2. Operazioni che richiedono cleanup garantito

**Esempio: operazioni su file con finally**

```php
public function processFile(string $filePath): array
{
    $handle = fopen($filePath, 'r');
    
    try {
        $data = [];
        while (($line = fgets($handle)) !== false) {
            $data[] = processLine($line);
        }
        return $data;
    } finally {
        // Garantisce che il file venga sempre chiuso
        if (is_resource($handle)) {
            fclose($handle);
        }
    }
}
```

### 3. Transazioni database complesse

**Esempio: operazioni multi-step con rollback**

```php
public function transferFunds(Account $from, Account $to, float $amount): void
{
    try {
        DB::beginTransaction();
        
        $from->debit($amount);
        $to->credit($amount);
        Transaction::create([...]);
        
        DB::commit();
    } catch (InsufficientFundsException $e) {
        DB::rollBack();
        throw $e; // Rilancia eccezioni di dominio
    } catch (\Exception $e) {
        DB::rollBack();
        Log::error('Fund transfer failed', ['exception' => $e]);
        throw new TransferException('Transfer failed', 0, $e);
    }
}
```

---

## Quando non usarlo

### 1. Validazione dei dati

**❌ Non usare try-catch per la validazione.**

**✅ Usare Form Request:** Laravel gestisce automaticamente `ValidationException` e restituisce risposte appropriate.

```php
// Nel controller
public function store(CreateUserRequest $request)
{
    // $request->validated() è già validato
    $user = User::create($request->validated());
    return response()->json($user, 201);
}
```

### 2. Autorizzazione e autenticazione

**❌ Non usare try-catch per autorizzazione.**

**✅ Usare Policy o Middleware:** Laravel gestisce automaticamente `AuthorizationException`.

```php
public function show(User $user)
{
    $this->authorize('view', $user);
    return response()->json($user);
}
```

### 3. Operazioni Eloquent standard

**❌ Non usare try-catch per operazioni semplici.**

**✅ Lasciare che Laravel gestisca automaticamente:** L'Exception Handler globale gestisce `ModelNotFoundException`.

```php
// Laravel gestisce automaticamente ModelNotFoundException
public function show($id)
{
    $user = User::findOrFail($id);
    return response()->json($user);
}
```

### 4. Operazioni che Laravel gestisce già

**Job Queue:** Laravel gestisce automaticamente le eccezioni nei job. Usa `$tries` e `failed()` per gestire i fallimenti.

**HTTP Client:** Laravel HTTP Client non lancia eccezioni per default. Usa `successful()`, `clientError()`, `serverError()` o `onError()`.

---

## Alternative Laravel

### 1. Helper `rescue()` (Laravel 5.5+)

Alternativa più pulita per operazioni semplici con fallback:

```php
// Invece di try-catch
$value = rescue(function () {
    return riskyOperation();
}, 'default');
```

### 2. Helper `report()`

Reporta errori senza interrompere l'esecuzione:

```php
try {
    validateValue($value);
    return true;
} catch (\Exception $e) {
    report($e); // Logga ma continua
    return false;
}
```

### 3. Exception Handler globale

Configura la gestione centralizzata in `bootstrap/app.php` (Laravel 11+) per personalizzare reporting e rendering delle eccezioni.

---

## Esempi Pratici

### Service con gestione errori appropriata

```php
class OrderService
{
    public function createOrder(array $data): Order
    {
        try {
            return DB::transaction(function () use ($data) {
                $order = Order::create($data);
                $order->items()->createMany($data['items']);
                $this->processPayment($order);
                return $order;
            });
        } catch (PaymentException $e) {
            throw $e; // Rilancia eccezioni di dominio
        } catch (QueryException $e) {
            Log::error('Database error creating order', ['exception' => $e]);
            throw new OrderCreationException('Failed to create order', 0, $e);
        }
    }
}
```

### Controller pulito che delega ai service

```php
class OrderController extends Controller
{
    public function store(CreateOrderRequest $request, OrderService $service)
    {
        // Validazione gestita da Form Request
        // Eccezioni gestite dall'Exception Handler globale
        $order = $service->createOrder($request->validated());
        return response()->json($order, 201);
    }
}
```

---

## Fonti e Riferimenti

### Documentazione Ufficiale

- **Laravel Error Handling**  
  [https://laravel.com/docs/errors](https://laravel.com/docs/errors)

- **Laravel Validation**  
  [https://laravel.com/docs/validation](https://laravel.com/docs/validation)

- **Laravel Authorization**  
  [https://laravel.com/docs/authorization](https://laravel.com/docs/authorization)

### Articoli e Guide

- **"Stop Using Try-Catch in Laravel Controllers"**  
  [https://corner.buka.sh/stop-using-try-catch-in-laravel-controllers-heres-the-better-way/](https://corner.buka.sh/stop-using-try-catch-in-laravel-controllers-heres-the-better-way/)

- **"How to Handle Exceptions in Laravel Without Try-Catch Everywhere"**  
  [https://laraveltips.com/how-to-handle-exceptions-in-laravel-without-try-catch-everywhere/](https://laraveltips.com/how-to-handle-exceptions-in-laravel-without-try-catch-everywhere/)

- **"Laravel Exception Handling Best Practices"**  
  [https://howik.com/laravel-exception-handling-best-practices](https://howik.com/laravel-exception-handling-best-practices)

- **"Stop Writing Try-Catch Like This in Laravel"**  
  [https://medium.com/@kamrankhalid06/stop-writing-try-catch-like-this-in-laravel-f8886da384c7](https://medium.com/@kamrankhalid06/stop-writing-try-catch-like-this-in-laravel-f8886da384c7)

### Video Tutorial

- **Laravel e PHP Try-Catch: Eccezioni VS Errori?**  
  [https://www.youtube.com/watch?v=NyPsSNrfYCo](https://www.youtube.com/watch?v=NyPsSNrfYCo)

### Best Practices Generali

- **PHP Exception Handling**  
  [https://www.php.net/manual/en/language.exceptions.php](https://www.php.net/manual/en/language.exceptions.php)

- **Clean Code - Error Handling**  
  Robert C. Martin - "Clean Code: A Handbook of Agile Software Craftsmanship"

---

> **Contribuisci:**  
> Se hai esempi, pattern o best practice da condividere, apri una PR e arricchisci questa guida!

