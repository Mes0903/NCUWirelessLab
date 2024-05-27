# Wireless-Lab2
* 實驗環境
    * 硬體版本 : Raspberry Pi 3 Model B+
    * 作業系統 : Raspbian 2018-06-27
    * LoRa : SX1276晶片
* 實驗目標
    * 使用OTAA模式傳輸資料
    * 並用MQTT取得資料後解碼
###### tags: `Wireless`

## :star: 實驗結果
1. 跑完OTAA傳輸模式的流程，分別截圖`join_ttn.py`,`send_ttn.py`的結果
2. 傳輸資料後用MQTT查看，並用base64解密截圖。
3. **以組為單位(一個人上傳即可)**，將這些截圖結果貼成一個word檔案，命名為`wireless-lab2-groupxx.docx`，並上傳到LMS作業區。

## OTAA模式送出 (join_ttn.py)
這份程式主要修改來自 [jeroennijhof](https://github.com/jeroennijhof/LoRaWAN) 的程式碼

1. 將程式碼載下來
```shell
$ git clone https://github.com/Ox7FFFFFFF/Wireless-Lab2
```

* 程式資料夾
   * LoRaWAN - 與LoRaWAN相關的程式
   * SX127x - 控制晶片的SPI程式
   * send_ttn.py - 資料送出
   * join_ttn.py - 送出Join封包
   * config.json - Personalized [Dev EUI,AppEUI,AppKey]配置

2. 在`join_ttn.py`填入deveui,appeui,appkey
    ```python=81
    # Init
    deveui = list(bytearray.fromhex('0000000000000011'))
    appeui = list(bytearray.fromhex('1234efc7104f1230'))
    appkey = list(bytearray.fromhex('a346b6faef2bd33c16fe9b1d8d47a11d'))
    devnonce = [randrange(256), randrange(256)]
    ```

3. 發送Join Request，裡面會帶入deveui,appeui,devnonce
    ```python=66
        def join(self):
            lorawan = LoRaWAN.new(appkey)
            lorawan.create(MHDR.JOIN_REQUEST, {'deveui': deveui, 
            'appeui': appeui, 'devnonce': devnonce})
            self.write_payload(lorawan.to_raw())
            self.set_mode(MODE.TX)
    ```

4. 送出uplink後，會跳到`on_tx_done`
    ```python=55
        def on_tx_done(self):
            self.clear_irq_flags(TxDone=1)
            print("TxDone")
            # 切換到rx
            self.set_mode(MODE.STDBY)
            self.set_dio_mapping([0,0,0,0,0,0])
            self.set_invert_iq(1)
            self.reset_ptr_rx()
            sleep(4)
            self.set_mode(MODE.RXCONT)
    ```

5. 當收到downlink時，會跳到`on_rx_done`
    ```python=18
        def on_rx_done(self):
            print("RxDone")

            self.clear_irq_flags(RxDone=1)
            payload = self.read_payload(nocheck=True)

            lorawan = LoRaWAN.new([], appkey)
            lorawan.read(payload)
            lorawan.get_payload()
            # 判斷格式是否為JOIN_ACCEPT
            if lorawan.get_mhdr().get_mtype() == MHDR.JOIN_ACCEPT:
                print("get mic: ",lorawan.get_mic())
                print("compute mic: ",lorawan.compute_mic())
                print("valid mic: ",lorawan.valid_mic())

                # 判斷downlink是不是自己的
                if lorawan.valid_mic():
                    devaddr = binary_array_to_hex(lorawan.get_devaddr())
                    nwskey = binary_array_to_hex(lorawan.derive_nwskey(devnonce))
                    appskey = binary_array_to_hex(lorawan.derive_appskey(devnonce))
                    print("devaddr:",devaddr)
                    print("nwskey :",nwskey)
                    print("appskey:",appskey)

                    # 將收到的devaddr,nwskey,appskey寫到檔案中
                    config = {'devaddr':devaddr,'nwskey':nwskey,'appskey':appskey,'fCnt':0}
                    data = json.dumps(config, sort_keys = True, indent = 4, separators=(',', ': '))
                    fp = open("config.json","w")
                    fp.write(data)
                    fp.close()
                    print("Join Accept")
                    sys.exit(0)

                else:
                    print("Fail to join!")
                    sys.exit(0)
    ```
6. 執行`join_ttn.py`
    ```shell
    $ python3 join_ttn.py
    ```
    ![](https://i.imgur.com/lhGBNv4.png)

7. Join成功會取得devaddr,nwkskey,appskey，並寫到`config.json`中
![](https://i.imgur.com/E9xfem2.png)

8. 使用上次的`send_ttn.py`傳輸資料，並使用MQTT查看傳輸的資料 
![](https://i.imgur.com/oWSamKA.png)
![](https://i.imgur.com/60Jyul7.png)
![](https://i.imgur.com/Unbb9oq.png)


