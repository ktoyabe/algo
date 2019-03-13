# SampleCode
```python
import unittest
import logging
from datetime import datetime

from back_tester.exchange import *
from back_tester.fwk.algorequest import AlgoInitRequest
from back_tester.runner import runner


class SampleAlgoUsingRunnerTestSuite(unittest.TestCase):
    """SampleAlgoUsingRunner test cases."""

    def setUp(self):
        self.back_tester = runner.Runner(datetime(2019, 3, 4, 8, 0, 0), datetime(2019, 3, 4, 8, 0, 0, 20 * 1000))
        self.back_tester.setup()
        self.back_tester.logger.loglevel = logging.DEBUG

    def test_algo_is_paused_by_fully_filled(self):
        params = dict()
        params["issue_code"] = "1570"
        params["venue"] = "TYO-MAIN"
        params['side'] = 'Buy'
        params['price'] = 14540
        params['qty'] = 900

        listed = self.back_tester.get_entity_manager().get_table("Listed").find(('1570', 'TYO-MAIN'))

        self.back_tester.register_orderbook(datetime(2019, 3, 4, 8, 0, 0, 8 * 1000),
                                            ExchangeOrderBook(listed, 14510, 1000, 14500, 200,
                                                              ExchangeStatus.TempDisabled,
                                                              datetime(2019, 3, 4, 8, 0, 0, 8 * 1000)))

        self.back_tester.register_orderbook(datetime(2019, 3, 4, 8, 0, 0, 9 * 1000),
                                            ExchangeOrderBook(listed, 14510, 1000, 14500, 200,
                                                              ExchangeStatus.Zaraba,
                                                              datetime(2019, 3, 4, 8, 0, 0, 9 * 1000)))

        self.back_tester.start(AlgoInitRequest("SampleAlgo", params))
```

# Output
```
2019-03-04 08:00:00.001000 DEBUG --- start ---
2019-03-04 08:00:00.001000 INFO on_init is called.
2019-03-04 08:00:00.001000 INFO on_start is called.
2019-03-04 08:00:00.001000 DEBUG --- end ---
2019-03-04 08:00:00.002000 DEBUG --- start ---
2019-03-04 08:00:00.002000 DEBUG --- end ---
2019-03-04 08:00:00.003000 DEBUG --- start ---
2019-03-04 08:00:00.003000 DEBUG --- end ---
2019-03-04 08:00:00.004000 DEBUG --- start ---
2019-03-04 08:00:00.004000 INFO on_order_status_change is called. order={internal_order_id:1,exchange_order_id:1,listed:1570.T,side:Side.Buy,time_in_force:TimeInForce.GFD,price:14540.0,qty:900,last_update_time:2019-03-04 08:00:00.004000,order_status:OrderStatus.Active}, send_result=SendResult(tx_msg_type=<TxMsgType.New: 0>, error_no=0, error_msg=None)
2019-03-04 08:00:00.004000 DEBUG --- end ---
2019-03-04 08:00:00.005000 DEBUG --- start ---
2019-03-04 08:00:00.005000 DEBUG --- end ---
2019-03-04 08:00:00.006000 DEBUG --- start ---
2019-03-04 08:00:00.006000 DEBUG --- end ---
2019-03-04 08:00:00.007000 DEBUG --- start ---
2019-03-04 08:00:00.007000 DEBUG --- end ---
2019-03-04 08:00:00.008000 DEBUG --- start ---
2019-03-04 08:00:00.008000 DEBUG --- end ---
2019-03-04 08:00:00.009000 DEBUG --- start ---
2019-03-04 08:00:00.009000 INFO on_orderbook_change. listed='1570.T', ask=1000@14510, bid=200@14500
2019-03-04 08:00:00.009000 DEBUG --- end ---
2019-03-04 08:00:00.010000 DEBUG --- start ---
2019-03-04 08:00:00.010000 INFO on_orderbook_change. listed='1570.T', ask=1000@14510, bid=200@14500
2019-03-04 08:00:00.010000 INFO on_order_filled is called. Order={internal_order_id:1,exchange_order_id:1,listed:1570.T,side:Side.Buy,time_in_force:TimeInForce.GFD,price:14540.0,qty:900,last_update_time:2019-03-04 08:00:00.004000,order_status:OrderStatus.Inactive}, Fill={exchange_order_id:1,price:14540.0,qty:900,exch_fill_time:2019-03-04 08:00:00.009000,fill_time:2019-03-04 08:00:00.010000}
2019-03-04 08:00:00.010000 INFO no active order. AlgoStatus: AlgoStatus.Running -> AlgoStatus.Paused
2019-03-04 08:00:00.010000 DEBUG --- end ---
2019-03-04 08:00:00.011000 DEBUG --- start ---
2019-03-04 08:00:00.011000 DEBUG --- end ---
2019-03-04 08:00:00.012000 DEBUG --- start ---
2019-03-04 08:00:00.012000 DEBUG --- end ---
2019-03-04 08:00:00.013000 DEBUG --- start ---
2019-03-04 08:00:00.013000 DEBUG --- end ---
2019-03-04 08:00:00.014000 DEBUG --- start ---
2019-03-04 08:00:00.014000 DEBUG --- end ---
2019-03-04 08:00:00.015000 DEBUG --- start ---
2019-03-04 08:00:00.015000 DEBUG --- end ---
2019-03-04 08:00:00.016000 DEBUG --- start ---
2019-03-04 08:00:00.016000 DEBUG --- end ---
2019-03-04 08:00:00.017000 DEBUG --- start ---
2019-03-04 08:00:00.017000 DEBUG --- end ---
2019-03-04 08:00:00.018000 DEBUG --- start ---
2019-03-04 08:00:00.018000 DEBUG --- end ---
2019-03-04 08:00:00.019000 DEBUG --- start ---
2019-03-04 08:00:00.019000 DEBUG --- end ---
2019-03-04 08:00:00.020000 DEBUG --- start ---
2019-03-04 08:00:00.020000 DEBUG --- end ---
```