C#でソケットサーバーの負荷テスト用クライアントの実装例
=================================================

ソケットサーバーの堅牢性テスト（負荷テスト）用クライアントですね。Windows環境でC#（.NET）を使用する場合、`Task` を活用した非同期並列処理で実装するのが最も効率的です。

大量の接続を維持しつつ、各接続が独立したタイミングで動くように `PeriodicTimer` とランダムな初期遅延を組み込んだコードを作成しました。

このプログラムはコンソールアプリケーションとして動作します。

```csharp
using System;
using System.Collections.Generic;
using System.Net.Sockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

class LoadTestClient
{
    // 設定項目
    private static string serverIp = "127.0.0.1";
    private static int serverPort = 8080;
    private static int connectionCount = 100; // 同時接続数
    private static int intervalMs = 1000;    // 送信間隔(ms)

    static async Task Main(string[] args)
    {
        Console.WriteLine("--- Socket Load Tester ---");
        
        var cts = new CancellationTokenSource();
        var tasks = new List<Task>();

        for (int i = 0; i < connectionCount; i++)
        {
            int clientId = i;
            // 各クライアントを非同期で起動
            tasks.Add(RunClientAsync(clientId, cts.Token));
            
            // 接続ラッシュによるサーバーダウンを防ぐため、接続開始自体も少しずらす
            await Task.Delay(10); 
        }

        Console.WriteLine($"{connectionCount} 個のクライアントが稼働中... Enterで終了");
        Console.ReadLine();
        cts.Cancel();

        await Task.WhenAll(tasks);
        Console.WriteLine("全クライアントを停止しました。");
    }

    static async Task RunClientAsync(int id, CancellationToken token)
    {
        try
        {
            using (var client = new TcpClient())
            {
                await client.ConnectAsync(serverIp, serverPort);
                using (var stream = client.GetStream())
                {
                    // 【重要】メッセージ送信タイミングをずらすための初期ランダム遅延
                    var random = new Random();
                    await Task.Delay(random.Next(0, intervalMs), token);

                    using var timer = new PeriodicTimer(TimeSpan.FromMilliseconds(intervalMs));
                    
                    while (await timer.WaitForNextTickAsync(token))
                    {
                        string message = $"Client {id}: Heartbeat at {DateTime.Now:HH:mm:ss.fff}\n";
                        byte[] data = Encoding.UTF8.GetBytes(message);
                        
                        await stream.WriteAsync(data, 0, data.Length, token);
                        // Console.WriteLine($"[Sent] {id}"); // デバッグ用（大量だと重くなるので注意）
                    }
                }
            }
        }
        catch (OperationCanceledException) { /* 正常終了 */ }
        catch (Exception ex)
        {
            Console.WriteLine($"Client {id} Error: {ex.Message}");
        }
    }
}

```

---

### 実装のポイント

* **`PeriodicTimer` (C# 6.0以降):** 従来の `Timer` よりも正確で、非同期ループに適しています。指定した `intervalMs` ごとに正確に実行されます。
* **初期遅延 (`Task.Delay`):** ご要望の「送信時刻をずらす」機能です。`random.Next(0, intervalMs)` を入れることで、全クライアントが同じタイミング（例：毎秒00ミリ秒）に一斉送信する「スパイク」現象を防ぎ、サーバーへの負荷を平滑化します。
* **非同期 I/O:** `ConnectAsync` や `WriteAsync` を使うことで、スレッドを無駄に消費せず、1つのプロセスで数千?数万の接続をシミュレート可能です。

### テスト時の注意点

1. **Windowsのエフェメラルポート制限:**
Windowsのデフォルト設定では、短時間に大量のTCP接続を行うとポート不足（16384上限など）になることがあります。数千単位のテストをする場合は、レジストリで `MaxUserPort` を調整する必要があるかもしれません。
2. **リソース監視:**
サーバー側の堅牢性を測る際は、CPU/メモリだけでなく **「ハンドル数（Handles）」** と **「TCP接続状態（ESTABLISHED）」** をタスクマネージャーやリソースモニターで観察してください。
