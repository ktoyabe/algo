# Sample Code

```
from typing import List, Optional
import datetime
import unittest

from back_tester.algo import algo
from back_tester.algo.algo import Acknowledger, Acknowledged, LogicContext, ParameterReader
from back_tester.sim.runner import AlgoRunner
from back_tester.fwk.algorequest import *
from back_tester.entity import entity
from back_tester.algo.order import Order, OrderStatus, SendResult, Fill
from back_tester.orderbook import orderbook
from back_tester import domain
from back_tester import exchange


class SampleAlgo(algo.Algo):
    def __init__(self):
        super().__init__()
        self.logic_context: Optional[LogicContext] = None
        self.listed: Optional[entity.Listed] = None
        self.side: Optional[domain.Side] = None
        self.time_in_force: Optional[domain.TimeInForce] = None
        self.price: Optional[float] = None
        self.qty: Optional[int] = None

        self.orders: List[Order] = []

    def ctor(self, logic_context: LogicContext):
        self.logic_context = logic_context

    def on_order_status_change(self, order: Order, result: SendResult):
        print("on_order_status_change is called. order={}, send_result={}".format(order, result))
        if self.status == algo.AlgoStatus.Running and not self._has_active_order():
            print("no active order. AlgoStatus: {} -> {}".format(algo.AlgoStatus.Running, algo.AlgoStatus.Paused))
            self.status = algo.AlgoStatus.Paused

    def on_order_filled(self, order: Order, fill: Fill):
        print("on_order_filled is called. Order={}, Fill={}".format(order, fill))
        if self.status == algo.AlgoStatus.Running and not self._has_active_order():
            print("no active order. AlgoStatus: {} -> {}".format(algo.AlgoStatus.Running, algo.AlgoStatus.Paused))
            self.status = algo.AlgoStatus.Paused

    def on_orderbook_change(self, ob: orderbook.OrderBook):
        print("on_orderbook_change. listed='{}', ask={}@{}, bid={}@{}".format(ob.listed.get_code(),
                                                                              ob.ask_size(0), ob.ask(0),
                                                                              ob.bid_size(0), ob.bid(0)))

    def _has_active_order(self):
        for o in self.orders:
            if o.order_status != OrderStatus.Inactive:
                return True

        return False

    def on_init(self, acknowledger: Acknowledger, param_reader: ParameterReader) -> Acknowledged:
        print("on_init is called.")

        # get listed
        issue_code = param_reader['issue_code']
        if issue_code is None:
            return acknowledger.error("issue_code is missing in param_reader.")
        venue = param_reader['venue']
        if venue is None:
            return acknowledger.error("venue is missing in param_reader.")
        listed_table = self.logic_context.entity_manager.get_table("Listed")
        if listed_table is None:
            return acknowledger.error("listed table is not found.")
        self.listed = listed_table.find((issue_code, venue))
        if self.listed is None:
            return acknowledger.error("listed is not found. issue_code='{}', venue='{}'".format(issue_code, venue))

        # side
        side_str = param_reader.get('side')
        if side_str is None:
            return acknowledger.error("side is missing in param_reader.")
        self.side = domain.str_to_side(side_str)
        if self.side is None:
            return acknowledger.error("unknown side. side_str='{}'".format(side_str))

        # time_in_force
        time_in_force_str = param_reader.get('time_in_force')
        if time_in_force_str is None:
            self.time_in_force = domain.TimeInForce.GFD
        else:
            self.time_in_force = domain.str_to_time_in_force(time_in_force_str)
            if self.time_in_force is None:
                return acknowledger.error("unknown time_in_foce. time_in_force_str='{}'".format(time_in_force_str))

        # price
        price_str = param_reader.get('price')
        if price_str is None:
            return acknowledger.error("price is missing in param_reader.")
        self.price = float(price_str)

        # qty
        qty_str = param_reader.get('qty')
        if qty_str is None:
            return acknowledger.error("qty is missing in param_reader.")
        self.qty = int(qty_str)

        self.logic_context.order_manager.add_order_status_change_listener(self.on_order_status_change)
        self.logic_context.order_manager.add_fill_listener(self.on_order_filled)
        self.logic_context.orderbook_manager.add_orderbook_listener(self.on_orderbook_change)

        self.status = algo.AlgoStatus.Initialized
        return acknowledger.ok()

    def on_start(self, acknowledger: Acknowledger) -> Acknowledged:
        print("on_start is called. listed='{}'".format(self.listed.get_code()))
        om = self.logic_context.order_manager

        order = om.new_order(self.listed, self.side, self.time_in_force, self.price, self.qty)
        if order is None:
            return acknowledger.error("failed to send order.")

        self.orders.append(order)
        self.status = algo.AlgoStatus.Running
        return acknowledger.ok()


class SampleAlgoFactory(algo.AlgoFactory):
    def create(self) -> algo.Algo:
        return SampleAlgo()


class SampleAlgoTestSuite(unittest.TestCase):
    """SampleAlgo test cases."""

    def setUp(self):
        from datetime import datetime
        from back_tester.entity import entity, entitymgr

        self.runner = AlgoRunner(datetime(2019, 3, 4, 8, 0, 0))
        venue_table: entity.EntityTable[str, entity.Venue] = \
            self.runner.fwk.entity_manager.get_or_create_table("Venue")
        listed_table: entity.EntityTable[[str, str], entity.Listed] = \
            self.runner.fwk.entity_manager.get_or_create_table("Listed")

        venue_tyo_main = entity.Venue('TYO-MAIN', '.T')
        venue_ose_main = entity.Venue('OSE-MAIN', '.OS')

        venue_table.insert(venue_tyo_main.name, venue_tyo_main)
        venue_table.insert(venue_ose_main.name, venue_ose_main)

        self.listed_1570 = entity.Listed('1570', venue_tyo_main)
        self.listed_fut_large = entity.Listed('164030018', venue_ose_main)

        listed_table.insert((self.listed_1570.issue_code, self.listed_1570.venue.name), self.listed_1570)
        listed_table.insert((self.listed_fut_large.issue_code, self.listed_fut_large.venue.name), self.listed_fut_large)

        for listed in listed_table.entities():
            self.runner.exchange.create_exchange_by_listed(listed)

        self.runner.fwk.register_algo_factory("SampleAlgo", SampleAlgoFactory())

    def perform_event_processing_ms(self, ms):
        self.runner.perform_event_processing(datetime.timedelta(milliseconds=ms))

    def test_algo_is_paused_by_fully_filled(self):
        params = dict()
        params["issue_code"] = "1570"
        params["venue"] = "TYO-MAIN"
        params['side'] = 'Buy'
        params['price'] = 14540
        params['qty'] = 900

        self.runner.fwk.rx_algo_request_queue.append(AlgoRequest(AlgoReuqestType.Init,
                                                                 AlgoInitReuqest("SampleAlgo", params)))
        self.perform_event_processing_ms(0)
        self.assertIsNotNone(self.runner.fwk.algo_instance)
        self.assertEqual(algo.AlgoStatus.Initialized, self.runner.fwk.algo_instance.status)

        self.runner.fwk.rx_algo_request_queue.append(AlgoRequest(AlgoReuqestType.Start, AlgoStartRequest()))
        self.perform_event_processing_ms(0)
        self.assertEqual(algo.AlgoStatus.Running, self.runner.fwk.algo_instance.status)

        self.perform_event_processing_ms(2)
        self.perform_event_processing_ms(1)

        self.runner.exchange.perform_orderbook(
            exchange.ExchangeOrderBook(self.listed_1570, 14510, 1000, 14500, 200,
                                       exchange.ExchangeStatus.Zaraba, self.runner.fwk.timemgr.get_time()))
        self.perform_event_processing_ms(1)
        self.assertEqual(algo.AlgoStatus.Paused, self.runner.fwk.algo_instance.status)
```

# Output

```
on_init is called.
on_start is called. listed='1570.T'
on_order_status_change is called. order={internal_order_id:1,exchange_order_id:1,listed:1570.T,side:Side.Buy,time_in_force:TimeInForce.GFD,price:14540.0,qty:900,last_update_time:2019-03-04 08:00:00.003000,order_status:OrderStatus.Active}, send_result=SendResult(tx_msg_type=<TxMsgType.New: 0>, error_no=0, error_msg=None)
on_orderbook_change. listed='1570.T', ask=1000@14510, bid=200@14500
on_order_filled is called. Order={internal_order_id:1,exchange_order_id:1,listed:1570.T,side:Side.Buy,time_in_force:TimeInForce.GFD,price:14540.0,qty:900,last_update_time:2019-03-04 08:00:00.003000,order_status:OrderStatus.Inactive}, Fill={exchange_order_id:1,price:14540.0,qty:900,exch_fill_time:2019-03-04 08:00:00.003000,fill_time:2019-03-04 08:00:00.004000}
no active order. AlgoStatus: AlgoStatus.Running -> AlgoStatus.Paused
```
