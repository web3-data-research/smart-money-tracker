# 如何获取token的实时信息

## 1. 获取token的metadata

以 0xeb90EaeB30B80fc37597acd5d3F70e850F6429Ac 为例

- 获取 token 的 name, symbol, total_supply, decimals 数据：https://docs.chainbase.com/reference/gettokenmetadata

- 获取该 token 的所有池子, transponse 
  ```sql
  SELECT * FROM ethereum.dex_pools WHERE '0xeb90EaeB30B80fc37597acd5d3F70e850F6429Ac' = ANY(token_addresses);  
  ``` 
  ```js
  fetch('https://api.transpose.io/sql', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-API-KEY': '{{TRANSPOSE_KEY}}'
    },
    body: JSON.stringify({
        'sql': 'SELECT * FROM ethereum.dex_pools WHERE \'0xeb90EaeB30B80fc37597acd5d3F70e850F6429Ac\' = ANY(token_addresses);',
        'parameters': {},
        'options': {}
    })
  });
  ```
  
- 获取前十大地址持仓数和持仓总数
  ```bash
  curl --request GET \
     --url 'https://api.chainbase.online/v1/token/top-holders?chain_id=1&contract_address=0xeb90EaeB30B80fc37597acd5d3F70e850F6429Ac&page=1&limit=11' \
     --header 'accept: application/json' \
     --header 'x-api-key: demo'
  ```

- 十五分钟内的新买入人数
  ```sql
  WITH
    nb AS (
        SELECT DISTINCT
            (origin_address) AS address
        FROM
            ethereum.dex_swaps
        WHERE
            TIMESTAMP > (NOW() - INTERVAL '15 minutes')
            AND to_token_address = '0xeb90EaeB30B80fc37597acd5d3F70e850F6429Ac'
    ),
    nbc AS (
        SELECT
            COUNT(address) AS new_buyer_count
        FROM
            nb
    ),
    nsc AS (
        SELECT
            COUNT(DISTINCT (origin_address)) AS new_seller_count
        FROM
            ethereum.dex_swaps
        WHERE
            TIMESTAMP > (NOW() - INTERVAL '15 minutes')
            AND from_token_address = '0xeb90EaeB30B80fc37597acd5d3F70e850F6429Ac'
    ),
    ob AS (
        SELECT DISTINCT
            (owner_address) AS address
        FROM
            ethereum.historical_token_owners
        WHERE
            TIMESTAMP <= (NOW() - INTERVAL '15 minutes')
            AND contract_address = '0xeb90EaeB30B80fc37597acd5d3F70e850F6429Ac'
    ),
    newJoinerCount AS (
        SELECT
            COUNT(address) AS new_holder_count
        FROM
            nb
        WHERE
            address NOT IN (
                SELECT
                    address
                FROM
                    ob
            )
    )
  SELECT
      nbc.*,
      newJoinerCount.*,
      nsc.*
  FROM
      nbc
      LEFT JOIN newJoinerCount ON TRUE
      LEFT JOIN nsc ON TRUE;
  ```