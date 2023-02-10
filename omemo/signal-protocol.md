* Double Ratchet 在[signal](https://github.com/signalapp/libsignal-protocol-java/blob/fde96d22004f32a391554e4991e4e1f0a14c2d50/java/src/main/java/org/whispersystems/libsignal/SessionCipher.java)
  协议的实现中会为每个session保留2000个key，用于乱序消息的解密。
  注意只保留向前（因为棘轮的特性就是只能往前一个方向滚动）的key。
  
  如当前设备本地已经滚到100号，此时200号消息先抵达，101-199还未收到，那轮子就会往前滚动到200，同时存储101-199的Key。
  
  但是如果此时99号消息再一次抵达，就会解密失败了，抛出DuplicateMessageException，因为前向安全性，一个消息只能被解密一次。

```java
  private MessageKeys getOrCreateMessageKeys(SessionState sessionState,
                                             ECPublicKey theirEphemeral,
                                             ChainKey chainKey, int counter)
      throws InvalidMessageException, DuplicateMessageException
  {
    if (chainKey.getIndex() > counter) {
      if (sessionState.hasMessageKeys(theirEphemeral, counter)) {
        return sessionState.removeMessageKeys(theirEphemeral, counter);
      } else {
        throw new DuplicateMessageException("Received message with old counter: " +
                                                chainKey.getIndex() + " , " + counter);
      }
    }

    if (counter - chainKey.getIndex() > 2000) {
      throw new InvalidMessageException("Over 2000 messages into the future!");
    }

    while (chainKey.getIndex() < counter) {
      MessageKeys messageKeys = chainKey.getMessageKeys();
      sessionState.setMessageKeys(theirEphemeral, messageKeys);
      chainKey = chainKey.getNextChainKey();
    }

    sessionState.setReceiverChainKey(theirEphemeral, chainKey.getNextChainKey());
    return chainKey.getMessageKeys();
  }
```

## reference

https://blog.jabberhead.tk/2019/04/04/shaking-hands-with-omemo-x3dh/

MAX_SKIP in https://xmpp.org/extensions/xep-0384.html

