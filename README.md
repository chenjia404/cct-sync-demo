### 说明
CCT同步demo


依赖三方库 [chenjia404/qk-php-sdk](https://github.com/chenjia404/qk-php-sdk)


### demo代码
```php
<?php
use quarkblockchain\QkNodeRPC;
use quarkblockchain\ERC20;
    /**
     * 同步官方托管地址的交易记录
     * @throws \Exception
     */
    public function syncTx()
    {
        $lastBlock = 0;//初始区块高度
        $num = 500;//一次同步500个区块
        for ($i = 0; $i < $num; $i++) {
            //组装参数
            if ($lastBlock < 10) {
                $blockArray[$i] = ['0x' . $lastBlock, true];
            } else {
                $blockArray[$i] = ['0x' . base_convert($lastBlock, 10, 16), true];
            }

            $lastBlock++;
        }

        try{
            $blocks = $this->rpc("eth_getBlockByNumber",$blockArray);
        }
        catch (\Exception $exception)
        {
            echo "请求接口超时 \n";
            exit;
        }
        DB::beginTransaction();
        try {

            if ($blocks) {
                echo "区块获取成功 \n";
                //循环区块
                foreach ($blocks as $block) {
                    if ($block['result']) {

                        $transactions = $block['result']['transactions'];
                        //如果此区块有交易，保存交易
                        if (isset($transactions) && count($transactions) > 0) {
                            $timestamp = date("Y-m-d H:i:s", base_convert($block['result']['timestamp'], 16, 10));
                            foreach ($transactions as $tx) {
                                //保存交易
                                saveTx($tx, $timestamp);
                            }
                        }

                        $block_height = bcadd(base_convert($block['result']['number'], 16, 10), 1, 0);
                    } else {
                        //如果没交易，保存已同步的最大区块高度，下一次就直接从这个高度之后开始同步
                        DB::commit();
                        echo "同步成功，当前高度:$lastBlock\n";
                        return false;
                    }
                }
            }

            //交易同步完成，保存已同步的最大区块高度，下一次就直接从这个高度之后开始同步
            DB::commit();
            echo "同步成功，当前高度:$block_height\n";
            return true;
        } catch (\Exception $e) {
            DB::rollback();
            echo $e->getMessage();
            return false;
        }
    }

    /**
     * 保存交易
     * @param $v
     * @param $timestamp
     */
    public function saveTx($v, $timestamp)
    {

        //查询交易是否成功
        $receipt = $this->rpc("eth_getTransactionReceipt", [[$v['hash']]]);
        if (isset($receipt[0]['result'])) {
            //判断交易是否成功
            if(isset($receipt[0]['result']['root']))
            {
                $tx_status = 1;
            }else{
                $tx_status = base_convert($receipt[0]['result']['status'], 16, 10);
            }
            if($tx_status != 1)
            {
                echo "{$v['hash']}交易失败\n";
                return true;
            }
        }else{
            echo "{$v['hash']}没有回执\n";
            return true;
        }
        //写入交易记录表
        $tx = new Transactions();
        $tx->from = $v['from'];
        $tx->to = $v['to'] ?? '';
        $tx->uid = 0;
        $tx->hash = $v['hash'];
        $tx->block_hash = $v['blockHash'];
        $tx->block_number = base_convert($v['blockNumber'], 16, 10);
        $tx->gas_price = bcdiv($this->HexDec2($v['gasPrice']), gmp_pow(10, 18), 18);
        $tx->amount = bcdiv($this->HexDec2($v['value']), gmp_pow(10, 18), 18);
        $tx->created_at = $timestamp;
        $tx->tx_status = 1;

        //input可能为空
        $input = $v['input'] ?? '';

        // input存在，且以0xa9059cbb开头的，必为通证交易
        if (substr($input, 0, 10) === '0xa9059cbb') {
            //xxxx为服务器运行的夸克区块链节点端口号，如果不是调用的当前服务器的节点，请填写所调用的服务器IP地址
            $url = "http://127.0.0.1:xxxx";
            $url_arr = parse_url($url);
            $geth = new QkNodeRPC($url_arr['host'], $url_arr['port']);
            $erc20 = new ERC20($geth);
            $token = $erc20->token($v['to']);
            $decimals = $token->decimals();
            if($decimals < 1)
                return true;
            //保存通证交易+
            $token_tx = new TransactionInputTransfer($input);
            //判断to是否为cct合约地址，如果是则添加
            if ($v['to'] == "XXXXXXXXXXXXXXX") {
                $token_tx_amount = bcdiv($this->HexDec2($token_tx->amount), gmp_pow(10,$decimals), 18);
                //是通证，保存通证信息
                $tx->token_tx_amount = $token_tx_amount;
                $tx->payee = $token_tx->payee;//保存接收地址
            } else {
                echo "{$v['hash']}不支持资产\n";
                return true;
            }
        }

        //如果转出地址、转入地址、接收地址为系统托管地址，则写入数据库
        if($tx->from == "xxxxxxxxxxx" || $tx->to == "xxxxxxxxxxx" || $tx->payee == "xxxxxxxxxxxx")
        {
            $tx->save();
        }

        return $tx;
    }

    /**
     * rpc
     * @param $method
     * @param $params
     * @return mixed
     */
    public function rpc($method,$params)
    {
        $param = array();
        foreach ($params as $key => $item)
        {
            $id = rand(1,100);
            $param[$key] = [
                'jsonrpc'=>"2.0",
                "method"=>$method,
                "params"=>$item,
                "id"=>$id
            ];
        }

        $param = json_encode($param);
        $data_str = $this->curlPost($param);
        $data = json_decode($data_str,true);

        return $data;
    }

    /**
     * post请求
     * @param $data
     * @return mixed
     */
    public function curlPost($data)
    {
        $url = "http://127.0.0.1";//你运行节点的服务器地址
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        // post数据
        curl_setopt($ch, CURLOPT_POST, 1);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array(
            'Content-Type: application/json'
        ));
        // post的变量
        curl_setopt($ch, CURLOPT_POSTFIELDS, $data);
        $output = curl_exec($ch);
        curl_close($ch);

        return $output;
    }

    /**
     * 16进制转10进制
     * @param string $hex
     * @return int|string
     */
    function HexDec2(string $hex)
    {
        $dec = 0;
        $len = strlen($hex);
        for ($i = 1; $i <= $len; $i++) {
            $dec = bcadd($dec, bcmul(strval(hexdec($hex[$i - 1])), bcpow('16', strval($len - $i))));
        }
        return $dec;
    }
?>
```
