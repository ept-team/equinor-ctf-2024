### Shop 5 CTF Challenge Writeup

**Solved by**: Morten Hauge

The website for this challenge is a simple webshop where we can buy and sell items. The goal is to purchase the EPT Fidget toy for $2500, but we can only afford a single item, Fun, priced at $99 with the note "Here you go, you are now having fun!".

We can register as many users as we like, but each user is only given an initial balance of $100.

### Walkthrough

The challenge is solved by selling orders that belong to other users in order to obtain multiple copies of an item:

1. Create a user, e.g. `test`
2. Purchase the Fun item
3. Log out
4. Create a new user, e.g. `test2`
5. Log in as your new user
6. Purchase the Fun item
7. Sell the Fun item and intercept the request
8. Modify the payload from `order_ids=2` to `order_ids=1&order_ids=2`
9. ???
10. Profit

After performing this set of actions, your new balance should be $199. You can rinse and repeat this action to get to $2500 and purchase the flag.

`Flag: {G0in6_1n_l0ops_to0_f45t_m4k3s_m3_d1zzy}`

Once you have obtain multiple copies of the Fun item, you can also just sell them in the shop normally and repurchase them in order to increase your balance.

Due to a bug in the program, only one of your items will be sold when you sell multiple items, but you will be awarded the balance for all items you attempted to sell, as if the sale went through.

### Explanation

The endpoint used for selling takes in a list of Order IDs and performs a call to a function called `stream_sell_orders`.

This function returns a list of functions, which accept a `User` object as their input. The User object is used to verify that an order being sold belongs to the user attempting to sell it.

Given the following `Order`s and simplified version of the function:

```python
def stream_sell_orders(orders):
    for order in orders:
        def sell_order():
            print("Selling order:", order)

        yield sell_order

for sell_operation in stream_sell_orders(
    orders=[
        Order(id=1, user_id=1, item_id=1),
        Order(id=2, user_id=2, item_id=1),
    ]
):
    sell_operation()
```

We might expect the following output:

```python
>>> Selling order: Order(id=1, user_id=1, item_id=1)
>>> Selling order: Order(id=2, user_id=2, item_id=1)
```

However the actual output is:

```python
>>> Selling order: Order(id=2, user_id=2, item_id=1)
>>> Selling order: Order(id=2, user_id=2, item_id=1)
```

Oh no! Users are able to sell their own order multiple times. The [following blogpost](https://eev.ee/blog/2011/04/24/gotcha-python-scoping-closures/) provides a good explanation for this behavior, but we'll examine it in the context of this task specifically.

What's happening is that `order` is bound to the scope of `stream_sell_orders`, not to the individual functions that are created. This is most easily seen with a little bit of inspection. Let's modify the source code again to further reduce noise and simplify things:

```python
from models import Order

def stream_sell_orders(orders: list[Order]):
    sell_order_funcs = []

    for order in orders:
        def sell_order():
            print("Sell order:", order)

        sell_order_funcs.append(sell_order)
    
    breakpoint()
    return sell_order_funcs

stream_sell_orders(
    orders = [
        Order(id=1, user_id=1, item_id=1),
        Order(id=2, user_id=2, item_id=1),
    ]
)
```

```python
(Pdb) locals()
{
    'orders': [
        Order(id=1, user_id=1, item_id=1),
        Order(id=2, user_id=2, item_id=1)
    ],
    'sell_order_funcs': [
        <function stream_sell_orders.<locals>.sell_order at 0x7f4eb09da340>,
        <function stream_sell_orders.<locals>.sell_order at 0x7f4eae3d1800>
    ],
    'sell_order': <function stream_sell_orders.<locals>.sell_order at 0x7f4eae3d1800>,
    'order': Order(id=2, user_id=2, item_id=1)
}
```

As we can see, in the scope of `stream_sell_orders`. The `order` variable, used by each of the functions, is always bound to the last order. This is the value that will be used when all of the functions are evaluated. As a result, if we attempt to sell orders that belong to other users, we are inadvertently allowed to sell our own orders _multiple times_.

Once we have obtained more than one copy of an item, we can use this bug to increase our balance without intercepting the request. If we sell orders 2 & 3 that both belong to us, only the last order (3) will be sold, but our balance will increase as if both items were sold correctly.

If you've paid close attention you might be wondering if the order we provide our order IDs matters when intercepting the initial request, e.g:

```
order_ids=1&order_ids=2
```

versus:

```
order_ids=2&order_ids=1
```

Luckily for us, there is a call to `list(set(order_ids))` in the original function to prevent payloads such as:

```
order_ids=2&order_ids=2
```

This has the consequence of sorting the list for us. As a result, the order in which we provide our IDs doesn't matter in the exploit payload.

#### A brief note on generators

Our simplified version of the source code uses a plain loop. The challenge is slightly different because it uses a generator function with `yield`. This would normally be a good way to avoid the scoping issue and behaves exactly as we would expect:

1. Start our loop
2. Enter `stream_sell_orders` scope
3. Create and yield function
4. Perform sell-operation
5. Re-enter `stream_sell_orders` scope with the next order
6. Create and yield function
7. Perform sell-operation

However, the call to `list` in `list(stream_sell_orders(...))` exhausts the generator immediately. This subtly changes the behavior and makes it equivalent to using a plain loop, causing the scoping issue.