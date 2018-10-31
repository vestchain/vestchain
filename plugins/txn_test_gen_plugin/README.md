# txn\_test\_gen\_plugin

This plugin provides a way to generate a given amount of transactions per second against the currency contract. It runs internally to vestd to reduce overhead.

This general procedure was used when doing Dawn 3.0 performance testing as mentioned in https://github.com/VESTIO/vest/issues/2078.

## Performance testing

The following instructions describe how to use the `txn_test_gen_plugin` plugin to generate 1,000 transaction per second load on a simple VESTIO node.

### Create config and data directories
Make an empty directory for our configs and data, `mkdir ~/vest.data`, and define a logging.json that doesn't print debug information (which occurs for each txn) to the console:
```bash
cat << EOF > ~/vest.data/logging.json
{
  "includes": [],
  "appenders": [{
      "name": "consoleout",
      "type": "console",
      "args": {
        "stream": "std_out",
        "level_colors": [{
            "level": "debug",
            "color": "green"
          },{
            "level": "warn",
            "color": "brown"
          },{
            "level": "error",
            "color": "red"
          }
        ]
      },
      "enabled": true
    }
  ],

  "loggers": [{
      "name": "default",
      "level": "info",
      "enabled": true,
      "additivity": false,
      "appenders": [
        "consoleout"
      ]
    }
  ]
}
EOF
```

### Launch producer
```bash
$ ./nodvest -d ~/vest.data/producer_node --config-dir ~/vest.data/producer_node -l ~/vest.data/logging.json --http-server-address "" -p vestio -e
```

### Launch non-producer that will generate transactions
```bash
$ ./nodvest -d ~/vest.data/generator_node --config-dir ~/vest.data/generator_node -l ~/vest.data/logging.json --plugin vestio::txn_test_gen_plugin --plugin vestio::wallet_api_plugin --plugin vestio::chain_api_plugin --p2p-peer-address localhost:9876 --p2p-listen-endpoint localhost:5555
```

### Create a wallet on the non-producer and set bios contract
```bash
$ ./clvest wallet create
$ ./clvest set contract vestio ~/vest/build.release/contracts/vestio.bios/ 
```

### Initialize the accounts txn_test_gen_plugin uses
```bash
$ curl --data-binary '["vestio", "5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3"]' http://localhost:8888/v1/txn_test_gen/create_test_accounts
```

### Start transaction generation, this will submit 20 transactions evey 20ms (total of 1000TPS)
```bash
$ curl --data-binary '["", 20, 20]' http://localhost:8888/v1/txn_test_gen/start_generation
```

### Note the producer console prints
```bash
vestio generated block 9b8b851d... #3219 @ 2018-04-25T16:07:47.000 with 500 trxs, lib: 3218
vestio generated block e5b3cd5d... #3220 @ 2018-04-25T16:07:47.500 with 500 trxs, lib: 3219
vestio generated block b243aeaa... #3221 @ 2018-04-25T16:07:48.000 with 500 trxs, lib: 3220
```

Note in the console output there are 500 transactions in each of the blocks which are produced every 500 ms yielding 1,000 transactions / second.
