# trans2quik
Library for importing transactions, entering orders into the QUIK ARQA Technologies trading system via the API.
#### Example of use
Cargo.toml
```
trans2quik = "1.0.0"
tracing = "0.1.40"
tracing-subscriber = "0.3.18"
lazy_static = "1.5.0"
```
main.rs
```
use lazy_static::lazy_static;
use std::error::Error;
use std::sync::{Arc, Condvar, Mutex};
use std::time::Duration;
use tracing::info;
use tracing_subscriber;
use trans2quik;

lazy_static! {
    static ref ORDER_CALLBACK_RECEIVED: Arc<(Mutex<bool>, Condvar)> =
        Arc::new((Mutex::new(false), Condvar::new()));
    static ref TRADE_CALLBACK_RECEIVED: Arc<(Mutex<bool>, Condvar)> =
        Arc::new((Mutex::new(false), Condvar::new()));
    static ref TRANSACTION_CALLBACK_RECEIVED: Arc<(Mutex<bool>, Condvar)> =
        Arc::new((Mutex::new(false), Condvar::new()));
}

fn main() -> Result<(), Box<dyn Error>> {
    tracing_subscriber::fmt::init();

    let path = r"c:\QUIK Junior\trans2quik.dll";
    let terminal = quik::Terminal::new(path)?;
    terminal.connect()?;
    terminal.is_dll_connected()?;
    terminal.is_quik_connected()?;
    terminal.set_connection_status_callback()?;
    terminal.set_transactions_reply_callback()?;
    let class_code = "QJSIM";
    let sec_code = "LKOH";
    terminal.subscribe_orders(class_code, sec_code)?;
    terminal.subscribe_trades(class_code, sec_code)?;
    terminal.start_orders();
    terminal.start_trades();
    let transaction_str = "ACCOUNT=NL0011100043; CLIENT_CODE=10677; TYPE=L; TRANS_ID=1; CLASSCODE=QJSIM; SECCODE=LKOH; ACTION=NEW_ORDER; OPERATION=B; PRICE=7103,5; QUANTITY=1;";
    terminal.send_async_transaction(transaction_str)?;

    // Waiting for callback or timeout
    {
        let order_received = {
            let (lock, cvar) = ORDER_CALLBACK_RECEIVED.as_ref();
            let received = lock.lock().unwrap();
            let timeout = Duration::from_secs(10);

            let (received, timeout_result) = cvar
                .wait_timeout_while(received, timeout, |received| !*received)
                .unwrap();

            if timeout_result.timed_out() {
                info!("Timed out waiting for order_status_callback");
            }

            *received
        };

        let trade_received = {
            let (lock, cvar) = TRADE_CALLBACK_RECEIVED.as_ref();
            let received = lock.lock().unwrap();
            let timeout = Duration::from_secs(10);

            let (received, timeout_result) = cvar
                .wait_timeout_while(received, timeout, |received| !*received)
                .unwrap();

            if timeout_result.timed_out() {
                info!("Timed out waiting for trade_status_callback");
            }

            *received
        };

        let transaction_received = {
            let (lock, cvar) = TRANSACTION_CALLBACK_RECEIVED.as_ref();
            let received = lock.lock().unwrap();
            let timeout = Duration::from_secs(10);

            let (received, timeout_result) = cvar
                .wait_timeout_while(received, timeout, |received| !*received)
                .unwrap();

            if timeout_result.timed_out() {
                info!("Timed out waiting for transaction_reply_callback");
            }

            *received
        };

        if !order_received && !trade_received && !transaction_received {
            info!("Did not receive all expected callbacks");
        }
    }

    terminal.unsubscribe_orders()?;
    terminal.unsubscribe_trades()?;
    terminal.disconnect()?;

    Ok(())
}
```