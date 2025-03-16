# trans2quik
Library for importing transactions, entering orders into the QUIK ARQA Technologies trading system via the API.
#### Example of use
```
pub async fn trade(
    mut command_receiver: mpsc::UnboundedReceiver<AppCommand>,
    instruments: Arc<RwLock<Vec<Instrument>>>,
) -> Result<(), Box<dyn Error + Send + Sync>> {
    let path = r"C:\QUIK\trans2quik.dll";
    let class_code = "";
    let sec_code = "";

    let terminal = Terminal::new(path)?;
    let terminal = Arc::new(Mutex::new(terminal));
    {
        let terminal_guard = terminal.lock().await;
        terminal_guard.connect()?;
        terminal_guard.is_dll_connected()?;
        terminal_guard.is_quik_connected()?;
        terminal_guard.set_connection_status_callback()?;
        terminal_guard.set_transactions_reply_callback()?;
        terminal_guard.subscribe_orders(class_code, sec_code)?;
        terminal_guard.subscribe_trades(class_code, sec_code)?;
        terminal_guard.start_orders();
        terminal_guard.start_trades();
    }

    let (transaction_sender, mut transaction_receiver): (
        UnboundedSender<TransactionInfo>,
        UnboundedReceiver<TransactionInfo>,
    ) = mpsc::unbounded_channel();

    {
        let mut transaction_reply_sender = TRANSACTION_REPLY_SENDER.lock().unwrap();
        *transaction_reply_sender = Some(transaction_sender);
    }

    let (order_sender, mut order_receiver): (
        UnboundedSender<OrderInfo>,
        UnboundedReceiver<OrderInfo>,
    ) = mpsc::unbounded_channel();

    {
        let mut order_status_sender = ORDER_STATUS_SENDER.lock().unwrap();
        *order_status_sender = Some(order_sender);
    }

    let (trade_sender, mut trade_receiver): (
        UnboundedSender<TradeInfo>,
        UnboundedReceiver<TradeInfo>,
    ) = mpsc::unbounded_channel();

    {
        let mut trade_status_sender = TRADE_STATUS_SENDER.lock().unwrap();
        *trade_status_sender = Some(trade_sender);
    }

    loop {
        tokio::select! {
            Some(command) = command_receiver.recv() => {
                match command {
                    AppCommand::Shutdown => {
                        info!("shutdown signal");
                        // Access to terminal via Mutex
                        let terminal_guard = terminal.lock().await;

                        if let Err(err) = terminal_guard.unsubscribe_orders() {
                            error!("error unsubscribing from orders: {}", err);
                        }
                        if let Err(err) = terminal_guard.unsubscribe_trades() {
                            error!("error unsubscribing from trades: {}", err);
                        }
                        if let Err(err) = terminal_guard.disconnect() {
                            error!("error disconnecting: {}", err);
                        }

                        info!("shutdown sequence completed");
                        break;
                    }
                }
            },
            Some(transaction_info) = transaction_receiver.recv() => {
                info!("transaction_reply_callback received: {:?}", transaction_info);
            },
            Some(order_info) = order_receiver.recv() => {
                info!("order_status_callback received: {:?}", order_info);
                if order_info.is_valid() {
                    // Do something with order info
                } else {
                    error!("order_info invalid");
                }
            },
            Some(trade_info) = trade_receiver.recv() => {
                info!("trade_status_callback received: {:?}", trade_info);
                if trade_info.is_valid() {
                    // Do something with trade info
                } else {
                    error!("trade_info invalid");
                }
            },
            result = async {
                if is_trading_time() {
                    let mut instruments = instruments.write().await;
                    for instrument in instruments.iter_mut() {
                            // Get access to the terminal
                            let terminal_guard = terminal.lock().await;
                            // Trade here
                    }
                } else {
                    info!("trading is paused, waiting for the next interval to check trading availability");
                }

                // Pause before the next iteration
                info!("sleep 60 seconds");
                sleep(Duration::from_secs(60)).await;

                Ok::<(), Box<dyn Error + Send + Sync>>(())
            } => {
                match result {
                    Ok(_) => {
                        // Everything went well, we continue the cycle
                    }
                    Err(err) => {
                        // Error Handling
                        error!("bot error: {}", err);
                        break;
                    }
                }
            },
        }
    }

    Ok(())
}
```