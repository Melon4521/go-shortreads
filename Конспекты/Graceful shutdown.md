---
created: "23/03/2026"
---
---

## Определение

**Graceful shutdown** - способность приложения корректно завершать работу. Например, закрыть соединения к базам данных или завершить тяжеловесные вычисления и успеть записать их на диск.

## Реализация

В Go слушать сигналы для завершения можно через пакет `os/signal`. Основные типы сигналов:

- **SIGINT (2)** - сигнал прерывания, обычно посылается при нажатии `Ctrl + C`.
- **SIGTERM (15**) - сигнал завершения. Это "вежливый" запрос на завершение, который дает приложению время на "уборку".
- **SIGKILL (9)** - сигнал принудительного завершения. Не может быть перехвачен или проигнорирован. Используется как крайняя мера.
- **SIGHUP (1)** - сигнал "повесить трубку". Исторически использовался для уведомления об обрыве соединения с терминалом. Сегодня часто используется для перезагрузки конфигурации.

```go
package main

import (
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // Создаём канал для приёма сигналов, буферезированный, чтобы точно положить сообщение в буфер и не потерять его
    sigChan := make(chan os.Signal, 1)
    
    // Указываем, какие сигналы мы хотим получать и куда будем писать при их появлении
    signal.Notify(sigChan,
        syscall.SIGINT,  // Ctrl+C
        syscall.SIGTERM, // Сигнал завершения от системы/Kubernetes
        syscall.SIGHUP,  // Сигнал перезапуска конфигурации
    )
    
    fmt.Println("Приложение запущено. Ожидаю сигналы...")
    
    // Запускаем фоновую работу
    go func() {
        for i := 1; i <= 10; i++ {
            fmt.Printf("Выполняю работу %d/10...\n", i)
            time.Sleep(2 * time.Second)
        }
    }()
    
    // Блокируемся до получения сигнала
    sig := <-sigChan
    log.Printf("Получен сигнал: %v (%v)\n", sig, sig)
}
```

## Пример для http-сервера

```go
func gracefulServer() {
    server := &http.Server{
        Addr:    ":8080",
        Handler: createRouter(),
    }
    
    // Запускаем сервер в горутине
    go func() {
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Printf("Ошибка сервера: %v", err)
        }
    }()
    
    // Ждём сигнала
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    <-sig
    
    log.Println("Получен сигнал завершения, начинаем graceful shutdown...")
    
    // Создаём контекст с тайм-аутом для shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    // Корректно завершаем работу
    if err := server.Shutdown(ctx); err != nil {
        log.Printf("Ошибка при shutdown: %v", err)
        server.Close() // Вынужденное завершение
    }
    
    log.Println("Сервер корректно завершён")
}
```