### Networking

In this lab, we will program on **NIC** driver to control packet receive and transmit process. Because the lab has a detailed description of how to do the lab, I'll just put on our code and explain some key points in the lab.



#### Code Part

```c
int
e1000_transmit(struct mbuf *m)
{
  //
  // Your code here.
  //
  // the mbuf contains an ethernet frame; program it into
  // the TX descriptor ring so that the e1000 sends it. Stash
  // a pointer so that it can be freed after sending.
  //
  acquire(&e1000_lock);
  int txport = regs[E1000_TDT];
  if(tx_ring[txport].status != E1000_TXD_STAT_DD){
    printf("haven't found a packet\n");
    release(&e1000_lock);
    return -1;
  }
  if(tx_mbufs[txport]){
    mbuffree(tx_mbufs[txport]);
  }
  tx_mbufs[txport] = m;
  tx_ring[txport].addr = (uint64)m->head;
  tx_ring[txport].length = m->len;
  tx_ring[txport].cmd = E1000_TXD_CMD_EOP | E1000_TXD_CMD_RS;

  regs[E1000_TDT] = (txport + 1) % TX_RING_SIZE;
  release(&e1000_lock);
  return 0;
}

static void
e1000_recv(void)
{
  //
  // Your code here.
  //
  // Check for packets that have arrived from the e1000
  // Create and deliver an mbuf for each packet (using net_rx()).
  //
  while(1){
    int next_idx = (regs[E1000_RDT] + 1) % RX_RING_SIZE;
    if(0 == (rx_ring[next_idx].status & E1000_RXD_STAT_DD)){
      break;
    }
    rx_mbufs[next_idx]->len = rx_ring[next_idx].length;
    net_rx(rx_mbufs[next_idx]);
    struct mbuf *new = mbufalloc(0);
    rx_ring[next_idx].addr = (uint64)new->head;
    rx_ring[next_idx].status = 0;
    rx_mbufs[next_idx] = new;
    regs[E1000_RDT] = next_idx;
  }
}
```



#### 