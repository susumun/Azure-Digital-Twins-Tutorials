# Azure-Digital-Twins-Tutorials
## Step01
### Azure Digital Twins インスタンスを準備し、基本的な動作を確認する

公式手順：https://docs.microsoft.com/ja-jp/azure/digital-twins/tutorial-code


1. 事前準備
    - .NET Core 3.1のダウンロード
        - https://dotnet.microsoft.com/download/dotnet-core/3.1
    - Azure Potral
        - https://portal.azure.com

2. Azure Digital Twins インスタンスを作成と、クライアントアプリのAzureADアプリ登録・権限付与
    - Azure Digital Twins インスタンスと認証を設定する (ポータル)
        - https://docs.microsoft.com/ja-jp/azure/digital-twins/how-to-set-up-instance-portal
    - Azure Digital Twins インスタンスと認証を設定する (CLI)
        - https://docs.microsoft.com/ja-jp/azure/digital-twins/how-to-set-up-instance-cli
    - Azure Digital Twins インスタンスと認証を設定する (スクリプト化)
        - https://docs.microsoft.com/ja-jp/azure/digital-twins/how-to-set-up-instance-scripted

    1. リソースグループを作成する
    2. Azure Digital Twins インスタンスを作成する
    3. ADTインスタンスにロールを割り当てる（ADT→アクセス制御(IAM)より）
        - Azure Digital Twins所有者ロール
    4. AzureADでクライアントアプリケーションを登録する
    5. AzureADで登録したアプリケーションのAPI許可を設定する

3. クライアントアプリのプロジェクト設定
    1. 空の.NETコンソールアプリプロジェクトを作成する
       アプリ用のフォルダに移動し、下記コマンドを実行する（cmd/sh）
        ```
        dotnet new console
        ```
    2. ADTを使用するための依存関係を2つ追加する
        ```
        dotnet add package Azure.DigitalTwins.Core --version 1.0.0-preview.3
        dotnet add package Azure.identity
        ```

4. クライアントアプリのコーディング
    - 公式手順に沿って実装
        - https://docs.microsoft.com/ja-jp/azure/digital-twins/tutorial-code#get-started-with-project-code
            ```
            using System;
            using Azure.DigitalTwins.Core;
            using Azure.Identity;
            using System.Threading.Tasks;
            using System.IO;
            using System.Collections.Generic;
            using Azure;
            using Azure.DigitalTwins.Core.Models;
            using System.Text.Json;
            using Azure.DigitalTwins.Core.Serialization;
                

            namespace AzureDigitalTwinsV2CodeTutorial
            {
                class Program
                {
                    //static void Main(string[] args)
                    static async Task Main(string[] args)
                    {
                        Console.WriteLine("Hello World!");

                        string clientId = "<your-application-ID>";
                        string tenantId = "<your-directory-ID>";
                        string adtInstanceUrl = "https://<your-Azure-Digital-Twins-instance-hostName>";
                        var credentials = new InteractiveBrowserCredential(tenantId, clientId);
                        DigitalTwinsClient client = new DigitalTwinsClient(new Uri(adtInstanceUrl), credentials);
                        Console.WriteLine($"Service client created – ready to go");

                        Console.WriteLine();
                        Console.WriteLine($"Upload a model");
                        var typeList = new List<string>();
                        string dtdl = File.ReadAllText("SampleModel.json");
                        typeList.Add(dtdl);
                        // Upload the model to the service
                        //await client.CreateModelsAsync(typeList);
                        try {
                            await client.CreateModelsAsync(typeList);
                        } catch (RequestFailedException rex) {
                            Console.WriteLine($"Load model: {rex.Status}:{rex.Message}");
                        }

                        // Read a list of models back from the service
                        AsyncPageable<ModelData> modelDataList = client.GetModelsAsync();
                        await foreach (ModelData md in modelDataList)
                        {
                            Console.WriteLine($"Type name: {md.DisplayName}: {md.Id}");
                        }

                        // Initialize twin data
                        BasicDigitalTwin twinData = new BasicDigitalTwin();
                        twinData.Metadata.ModelId = "dtmi:com:contoso:SampleModel;1";
                        twinData.CustomProperties.Add("data", $"Hello World!");

                        string prefix="sampleTwin-";
                        for(int i=0; i<3; i++) {
                            try {
                                twinData.Id = $"{prefix}{i}";
                                await client.CreateDigitalTwinAsync($"{prefix}{i}", JsonSerializer.Serialize(twinData));
                                Console.WriteLine($"Created twin: {prefix}{i}");
                            } catch(RequestFailedException rex) {
                                Console.WriteLine($"Create twin error: {rex.Status}:{rex.Message}");  
                            }
                        }

                        // Connect the twins with relationships
                        await CreateRelationship(client, "sampleTwin-0", "sampleTwin-1");
                        await CreateRelationship(client, "sampleTwin-0", "sampleTwin-2");

                        //List the relationships
                        await ListRelationships(client, "sampleTwin-0");

                        // Run a query    
                        AsyncPageable<string> result = client.QueryAsync("Select * From DigitalTwins");
                        await foreach (string twin in result)
                        {
                            object jsonObj = JsonSerializer.Deserialize<object>(twin);
                            string prettyTwin = JsonSerializer.Serialize(jsonObj, new JsonSerializerOptions { WriteIndented = true });
                            Console.WriteLine(prettyTwin);
                            Console.WriteLine("---------------");
                        }

                    }
                    public async static Task CreateRelationship(DigitalTwinsClient client, string srcId, string targetId)
                    {
                        var relationship = new BasicRelationship
                        {
                            TargetId = targetId,
                            Name = "contains"
                        };

                        try
                        {
                            string relId = $"{srcId}-contains->{targetId}";
                            await client.CreateRelationshipAsync(srcId, relId, JsonSerializer.Serialize(relationship));
                            Console.WriteLine("Created relationship successfully");
                        }
                        catch (RequestFailedException rex) {
                            Console.WriteLine($"Create relationship error: {rex.Status}:{rex.Message}");
                        }
                    }

                    public async static Task ListRelationships(DigitalTwinsClient client, string srcId)
                    {
                        try {
                            AsyncPageable<string> results = client.GetRelationshipsAsync(srcId);
                            Console.WriteLine($"Twin {srcId} is connected to:");
                            await foreach (string rel in results)
                            {
                                var brel = JsonSerializer.Deserialize<BasicRelationship>(rel);
                                Console.WriteLine($" -{brel.Name}->{brel.TargetId}");
                            }
                        } catch (RequestFailedException rex) {
                            Console.WriteLine($"Relationship retrieval error: {rex.Status}:{rex.Message}");   
                        }
                    }
                }
            }

            ```


5. 実行結果（本セクション完了時）
    ```
    Hello World!
    Service client created ? ready to go

    Upload a model
    Load model: 409:Service request failed.
    Status: 409 (Conflict)

    Content:
    {"error":{"code":"ModelAlreadyExists","message":"Model with same ID already exists dtmi:com:contoso:SampleModel;1. Use Model_List API to view models that already exist. See the Swagger example.(http://aka.ms/ModelListSwSmpl)"}}

    Headers:
    Strict-Transport-Security: REDACTED
    Date: Thu, 13 Aug 2020 07:38:50 GMT
    Content-Length: 227
    Content-Type: application/json; charset=utf-8

    Type name: System.Collections.Generic.Dictionary`2[System.String,System.String]: dtmi:com:contoso:SampleModel;1
    Type name: System.Collections.Generic.Dictionary`2[System.String,System.String]: dtmi:example:Room;2
    Type name: System.Collections.Generic.Dictionary`2[System.String,System.String]: dtmi:example:Floor;1
    Type name: System.Collections.Generic.Dictionary`2[System.String,System.String]: dtmi:contosocom:DigitalTwins:Thermostat;1
    Type name: System.Collections.Generic.Dictionary`2[System.String,System.String]: dtmi:contosocom:DigitalTwins:Space;1
    Created twin: sampleTwin-0
    Created twin: sampleTwin-1
    Created twin: sampleTwin-2
    Created relationship successfully
    Created relationship successfully
    Twin sampleTwin-0 is connected to:
    -contains->sampleTwin-1
    -contains->sampleTwin-2
    {
        "$dtId": "floor1",
        "$etag": "W/\u00224c40bbed-7253-4f75-8ba8-f564aa331a75\u0022",
        "DisplayName": "Floor 1",
        "Location": "Puget Sound",
        "Temperature": 65,
        "ComfortIndex": 0,
        "$metadata": {
            "$model": "dtmi:contosocom:DigitalTwins:Space;1",
            "DisplayName": {
            "desiredValue": "Floor 1",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "Location": {
            "desiredValue": "Puget Sound",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "Temperature": {
            "desiredValue": 65,
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "ComfortIndex": {
            "desiredValue": 0,
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            }
        }
    }
    ---------------
    {
        "$dtId": "sampleTwin-0",
        "$etag": "W/\u002213e093a2-f712-4724-ad8b-236f6029de67\u0022",
        "data": "Hello World!",
        "$metadata": {
            "$model": "dtmi:com:contoso:SampleModel;1",
            "data": {
            "desiredValue": "Hello World!",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            }
        }
    }
    ---------------
    {
        "$dtId": "room21",
        "$etag": "W/\u002216750b09-a44d-4a7d-83fb-14b05bf3e944\u0022",
        "DisplayName": "Room 21",
        "Location": "Puget Sound",
        "Temperature": 65,
        "ComfortIndex": 0,
        "$metadata": {
            "$model": "dtmi:contosocom:DigitalTwins:Space;1",
            "DisplayName": {
            "desiredValue": "Room 21",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "Location": {
            "desiredValue": "Puget Sound",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "Temperature": {
            "desiredValue": 65,
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "ComfortIndex": {
            "desiredValue": 0,
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            }
        }
    }
    ---------------
    {
        "$dtId": "thermostat67",
        "$etag": "W/\u00221bdf0288-0955-4cb9-8b79-6bc2626d281b\u0022",
        "DisplayName": "Thermostat 67",
        "Location": "Puget Sound",
        "FirmwareVersion": "1.3.9",
        "Temperature": 65,
        "ComfortIndex": 0,
        "$metadata": {
            "$model": "dtmi:contosocom:DigitalTwins:Thermostat;1",
            "DisplayName": {
            "desiredValue": "Thermostat 67",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "Location": {
            "desiredValue": "Puget Sound",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "FirmwareVersion": {
            "desiredValue": "1.3.9",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "Temperature": {
            "desiredValue": 65,
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            },
            "ComfortIndex": {
            "desiredValue": 0,
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            }
        }
    }
    ---------------
    {
        "$dtId": "sampleTwin-1",
        "$etag": "W/\u00227d028f04-4142-40ab-9c8e-543c0453129f\u0022",
        "data": "Hello World!",
        "$metadata": {
            "$model": "dtmi:com:contoso:SampleModel;1",
            "data": {
            "desiredValue": "Hello World!",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            }
        }
    }
    ---------------
    {
        "$dtId": "sampleTwin-2",
        "$etag": "W/\u002276ed9282-0c66-43fb-a5ba-ff62cf998264\u0022",
        "data": "Hello World!",
        "$metadata": {
            "$model": "dtmi:com:contoso:SampleModel;1",
            "data": {
            "desiredValue": "Hello World!",
            "desiredVersion": 1,
            "ackVersion": 1,
            "ackCode": 200,
            "ackDescription": "Auto-Sync"
            }
        }
    }
    ---------------
    ```

### Step01 Completed！