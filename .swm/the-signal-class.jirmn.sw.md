---
title: The Signal class
---
This document will provide an overview of the <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> class in <SwmPath>[src/kdbindings/signal.h](src/kdbindings/signal.h)</SwmPath>. We will cover:

1. What <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> is and what it is used for.
2. The variables and functions defined in the <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> class.

# What is Signal

The <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> class in <SwmPath>[src/kdbindings/signal.h](src/kdbindings/signal.h)</SwmPath> provides a mechanism for communication between objects. It recreates Qt's Signals & Slots mechanism in pure C++17. A <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> can be used to notify any number of slots that a certain event has occurred. The slot can be almost any callable object, including member functions and lambdas. This connection happens in a <SwmToken path="src/kdbindings/signal.h" pos="51:13:15" line-data=" * This connection happens in a type-safe manner, as a slot can only be connected to">`type-safe`</SwmToken> manner, as a slot can only be connected to a <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> when the arguments of the slot match the values the <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> emits. The <SwmToken path="src/kdbindings/signal.h" pos="337:13:13" line-data="    KDBINDINGS_WARN_UNUSED ConnectionHandle connect(std::function&lt;void(Args...)&gt; const &amp;slot)">`Args`</SwmToken> type parameter pack describes which value types the <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> will emit. Additionally, <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> supports deferred connections, enabling the decoupling of signal emission from the execution of connected slots.

<SwmSnippet path="/src/kdbindings/signal.h" line="337">

---

The function <SwmToken path="src/kdbindings/signal.h" pos="337:5:5" line-data="    KDBINDINGS_WARN_UNUSED ConnectionHandle connect(std::function&lt;void(Args...)&gt; const &amp;slot)">`connect`</SwmToken> is used to connect a <SwmToken path="src/kdbindings/signal.h" pos="337:7:9" line-data="    KDBINDINGS_WARN_UNUSED ConnectionHandle connect(std::function&lt;void(Args...)&gt; const &amp;slot)">`std::function`</SwmToken> to the signal. When <SwmToken path="src/kdbindings/signal.h" pos="327:5:7" line-data="     * When emit() is called on the Signal, the functions will be called with">`emit()`</SwmToken> is called on the <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken>, the functions will be called with the arguments provided to <SwmToken path="src/kdbindings/signal.h" pos="327:5:7" line-data="     * When emit() is called on the Signal, the functions will be called with">`emit()`</SwmToken>. The returned value can be used to disconnect the function again.

```c
    KDBINDINGS_WARN_UNUSED ConnectionHandle connect(std::function<void(Args...)> const &slot)
    {
        ensureImpl();

        return ConnectionHandle{ m_impl, m_impl->connect(slot) };
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="358">

---

The function <SwmToken path="src/kdbindings/signal.h" pos="358:5:5" line-data="    KDBINDINGS_WARN_UNUSED ConnectionHandle connectReflective(std::function&lt;void(ConnectionHandle &amp;, Args...)&gt; const &amp;slot)">`connectReflective`</SwmToken> establishes a connection between a signal and a slot, allowing the slot to access and manage its own connection handle. This method is particularly useful for creating connections that can autonomously manage themselves, such as disconnecting after being triggered a certain number of times or under specific conditions.

```c
    KDBINDINGS_WARN_UNUSED ConnectionHandle connectReflective(std::function<void(ConnectionHandle &, Args...)> const &slot)
    {
        ensureImpl();

        return ConnectionHandle{ m_impl, m_impl->connectReflective(slot) };
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="377">

---

The function <SwmToken path="src/kdbindings/signal.h" pos="377:5:5" line-data="    KDBINDINGS_WARN_UNUSED ConnectionHandle connectSingleShot(std::function&lt;void(Args...)&gt; const &amp;slot)">`connectSingleShot`</SwmToken> establishes a <SwmToken path="src/kdbindings/signal.h" pos="366:7:9" line-data="     * Establishes a single-shot connection between a signal and a slot and when the signal is emitted, the connection will be">`single-shot`</SwmToken> connection between a signal and a slot. When the signal is emitted, the connection will be disconnected and the slot will be called. Note that the slot will be disconnected before it is called.

```c
    KDBINDINGS_WARN_UNUSED ConnectionHandle connectSingleShot(std::function<void(Args...)> const &slot)
    {
        return connectReflective([slot](ConnectionHandle &handle, Args... args) {
            handle.disconnect();
            slot(args...);
        });
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="407">

---

The function <SwmToken path="src/kdbindings/signal.h" pos="407:5:5" line-data="    KDBINDINGS_WARN_UNUSED ConnectionHandle connectDeferred(const std::shared_ptr&lt;ConnectionEvaluator&gt; &amp;evaluator, std::function&lt;void(Args...)&gt; const &amp;slot)">`connectDeferred`</SwmToken> establishes a deferred connection between the provided evaluator and slot. This function allows connecting an evaluator and a slot such that the slot's execution is deferred until the conditions evaluated by the <SwmToken path="src/kdbindings/signal.h" pos="407:17:17" line-data="    KDBINDINGS_WARN_UNUSED ConnectionHandle connectDeferred(const std::shared_ptr&lt;ConnectionEvaluator&gt; &amp;evaluator, std::function&lt;void(Args...)&gt; const &amp;slot)">`evaluator`</SwmToken> are met.

```c
    KDBINDINGS_WARN_UNUSED ConnectionHandle connectDeferred(const std::shared_ptr<ConnectionEvaluator> &evaluator, std::function<void(Args...)> const &slot)
    {
        ensureImpl();

        ConnectionHandle handle(m_impl, {});
        handle.setId(m_impl->connectDeferred(evaluator, slot));
        return handle;
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="470">

---

The function <SwmToken path="src/kdbindings/signal.h" pos="470:3:3" line-data="    void disconnect(const ConnectionHandle &amp;handle)">`disconnect`</SwmToken> is used to disconnect a previously connected slot. After the slot was successfully disconnected, the <SwmToken path="src/kdbindings/signal.h" pos="470:7:7" line-data="    void disconnect(const ConnectionHandle &amp;handle)">`ConnectionHandle`</SwmToken> will no longer be active.

```c
    void disconnect(const ConnectionHandle &handle)
    {
        if (m_impl && handle.belongsTo(*this) && handle.m_id.has_value()) {
            m_impl->disconnect(handle);
            // TODO check if Impl is now empty and reset
        } else {
            throw std::out_of_range("Provided ConnectionHandle does not match any connection\nLikely the connection was deleted before!");
        }
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="489">

---

The function <SwmToken path="src/kdbindings/signal.h" pos="489:3:3" line-data="    void disconnectAll() noexcept">`disconnectAll`</SwmToken> disconnects all previously connected functions. All currently active <SwmToken path="src/kdbindings/signal.h" pos="78:23:23" line-data="    // This allows us to easily move Signals without losing their ConnectionHandles, as well as">`ConnectionHandles`</SwmToken> that belong to this <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> will no longer be active afterwards.

```c
    void disconnectAll() noexcept
    {
        if (m_impl) {
            m_impl->disconnectAll();
            // Once all connections are disconnected, we can release ownership of the Impl.
            // This does not destroy the Signal itself, just the Impl object.
            // If another slot is connected, another Impl object will be constructed.
            m_impl.reset();
        }
        // If m_impl is nullptr, we don't have any connections to disconnect
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="518">

---

The function <SwmToken path="src/kdbindings/signal.h" pos="518:3:3" line-data="    bool blockConnection(const ConnectionHandle &amp;handle, bool blocked)">`blockConnection`</SwmToken> sets the block state of the connection. If a connection is blocked, emitting the <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken> will no longer call this connection's slot until the connection is unblocked.

```c
    bool blockConnection(const ConnectionHandle &handle, bool blocked)
    {
        if (m_impl && handle.belongsTo(*this) && handle.m_id.has_value()) {
            return m_impl->blockConnection(*handle.m_id, blocked);
        } else {
            throw std::out_of_range("Provided ConnectionHandle does not match any connection\nLikely the connection was deleted before!");
        }
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="536">

---

The function <SwmToken path="src/kdbindings/signal.h" pos="536:3:3" line-data="    bool isConnectionBlocked(const ConnectionHandle &amp;handle) const">`isConnectionBlocked`</SwmToken> checks whether the connection is currently blocked. To change the blocked state of a connection, call <SwmToken path="src/kdbindings/signal.h" pos="530:22:24" line-data="     * To change the blocked state of a connection, call blockConnection().">`blockConnection()`</SwmToken>.

```c
    bool isConnectionBlocked(const ConnectionHandle &handle) const
    {
        assert(handle.belongsTo(*this));
        if (!m_impl) {
            throw std::out_of_range("Provided ConnectionHandle does not match any connection\nLikely the connection was deleted before!");
        }

        if (handle.m_id.has_value()) {
            return m_impl->isConnectionBlocked(*handle.m_id);
        } else {
            return false;
        }
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="568">

---

The function <SwmToken path="src/kdbindings/signal.h" pos="568:3:3" line-data="    void emit(Args... p) const">`emit`</SwmToken> emits the <SwmToken path="src/kdbindings/signal.h" pos="494:13:13" line-data="            // This does not destroy the Signal itself, just the Impl object.">`Signal`</SwmToken>, which causes all connected slots to be called, as long as they are not blocked. The arguments provided to <SwmToken path="src/kdbindings/signal.h" pos="568:3:3" line-data="    void emit(Args... p) const">`emit`</SwmToken> will be passed to each slot by copy.

```c
    void emit(Args... p) const
    {
        if (m_impl)
            m_impl->emit(p...);

        // if m_impl is nullptr, we don't have any slots connected, don't bother emitting
    }
```

---

</SwmSnippet>

# Usage

<SwmSnippet path="/src/kdbindings/connection_evaluator.h" line="110">

---

The <SwmToken path="src/kdbindings/connection_evaluator.h" pos="112:5:5" line-data="    friend class Signal;">`Signal`</SwmToken> class is used to manage connections and slot invocations. It provides a mechanism to enqueue slot invocations, ensuring that the appropriate functions are called when a signal is emitted.

```c
private:
    template<typename...>
    friend class Signal;

    void enqueueSlotInvocation(const ConnectionHandle &handle, const std::function<void()> &slotInvocation)
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN" repo-name="KDBindings"><sup>Powered by [Swimm](/)</sup></SwmMeta>
