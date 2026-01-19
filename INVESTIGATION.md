# Investigation Findings

iOS system logs captured via Console.app on iPhone 16 Pro running iOS 26.2.

## Test 1: Concurrent Appends (FAILS)

Two `appendBuffer()` calls made simultaneously on cold start.

```
01:48:35.027693  MobileSafari     MediaSource::setLogIdentifier(4443398812334050190)
01:48:35.028297  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

01:48:35.030852  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
01:48:35.032083  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

01:48:35.032102  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20
01:48:35.032113  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20

01:48:35.033664  mediaparserd     activating connection: com.apple.coremedia.mediaparserd.manifold.xpc.peer[29890]
01:48:35.038604  runningboardd    [mediaparserd] is not RunningBoard jetsam managed
01:48:35.038609  runningboardd    [mediaparserd] This process will not be managed

01:48:35.043497  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) OK
01:48:35.043560  mediaparserd     ERROR: FigApplicationStateMonitor signalled err=-19431
01:48:35.043833  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) ParsingFailed
01:48:35.043842  MobileSafari     MediaSource::streamEndedWithError decode
01:48:35.043869  MobileSafari     MediaSource::onReadyStateChange old state = open, new state = ended
```

**Result: SB1 OK, SB2 ParsingFailed**

---

## Test 2: Sequential Appends (SUCCESS)

Appends made one after another (wait for first to complete before second).

```
01:51:18.483222  MobileSafari     MediaSource::setLogIdentifier(12367319609071158855)
01:51:18.484461  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

01:51:18.503264  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
01:51:18.507823  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

01:51:18.507967  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20

01:51:18.513077  mediaparserd     activating connection: com.apple.coremedia.mediaparserd.manifold.xpc.peer[29935]
01:51:18.516253  runningboardd    [mediaparserd] is not RunningBoard jetsam managed
01:51:18.516272  runningboardd    [mediaparserd] This process will not be managed
01:51:18.520112  mediaparserd     ERROR: FigApplicationStateMonitor signalled err=-19431

01:51:18.525048  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) OK
01:51:18.525055  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20
01:51:18.525861  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) OK
```

**Result: Both OK (despite err=-19431)**

---

## Test 3: Two MMS Instances (First FAILS, Second SUCCESS)

Create MMS, fail, create another MMS.

First MMS:

```
01:51:44.151010  MobileSafari     MediaSource::setLogIdentifier(16352574382871214077)
01:51:44.153045  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

01:51:44.158086  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
01:51:44.158123  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

01:51:44.158217  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20
01:51:44.158302  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20

01:51:44.162900  mediaparserd     activating connection: com.apple.coremedia.mediaparserd.manifold.xpc.peer[29945]
01:51:44.171897  runningboardd    [mediaparserd] is not RunningBoard jetsam managed
01:51:44.171921  runningboardd    [mediaparserd] This process will not be managed
01:51:44.172019  mediaparserd     ERROR: FigApplicationStateMonitor signalled err=-19431

01:51:44.172513  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) OK
01:51:44.172939  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) ParsingFailed
01:51:44.172957  MobileSafari     MediaSource::streamEndedWithError decode
01:51:44.173115  MobileSafari     MediaSource::onReadyStateChange old state = open, new state = ended
```

Second MMS (created after first fails):

```
01:51:44.173566  MobileSafari     MediaSource::setLogIdentifier(16352574382871214077)
01:51:44.174032  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

01:51:44.174802  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
01:51:44.175064  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

01:51:44.175082  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20
01:51:44.175106  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20

01:51:44.175312  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) OK
01:51:44.175318  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) OK
```

**Result: First MMS - SB2 OK, SB1 ParsingFailed. Second MMS - Both OK**

---

## Test 4: Two Separate MMS with Single SourceBuffer Each (FAILS)

Two video elements, two MMS instances, each with only ONE SourceBuffer.

Setup:

- Two separate video elements
- Two separate ManagedMediaSource instances (one per video)
- Each MMS has only 1 SourceBuffer (video codec)
- Wait for both MMS to fire `sourceopen`
- Append 20-byte ftyp box to both SourceBuffers simultaneously

Result on cold start:

- MMS1: OK
- MMS2: ParsingFailed, readyState=ended

**Result: MMS1 OK, MMS2 ParsingFailed**

**Conclusion: The race is NOT between SourceBuffers within a single MMS. It happens at some shared layer between separate MMS instances.**

---

## Test 5: Two MMS, Single Append (SUCCESS)

Two MMS with SourceBuffers created, but only append to one.

Setup:

- Two separate video elements
- Two separate ManagedMediaSource instances
- Each MMS has 1 SourceBuffer
- Wait for both MMS to fire `sourceopen`
- Only append to MMS1 (MMS2 SourceBuffer exists but no append)

```
[0.043s] MMS1: sourceopen
[0.043s] MMS2: sourceopen
[0.043s] Both MMS open, creating SourceBuffers...
[0.047s] Created SB in both MMS, but only appending to MMS1...
[0.056s] MMS1: updateend, readyState=open
[0.056s] SUCCESS: Single append with 2 MMS works!
```

**Result: SUCCESS**

**Conclusion: Creating multiple MMS/SourceBuffers is fine. The race is specifically triggered by concurrent `appendBuffer` calls.**

---

## Analysis

### What We Know

1. **The race is specifically triggered by concurrent `appendBuffer` calls** - creating multiple MMS/SourceBuffers without appending is fine (Test 5)

2. **Sequential appends work** - even on cold start (Test 2)

3. **The mediaparserd error -19431 is NOT the cause** - it occurs in ALL tests, including successful ones

4. **It's not deterministic which request fails** - Test 1: SB2 failed. Test 3: SB1 failed. Test 4: MMS2 failed

5. **The race is NOT specific to SourceBuffers within a single MMS** - Test 4 proves two separate MMS instances still race

6. **The race happens at some shared layer** - something shared between separate MMS instances causes the failure

7. **Second/subsequent requests work** - once initialization completes, concurrent requests succeed

### Timeline Pattern

| Test          | Setup            | Append Pattern                      | Result                 |
| ------------- | ---------------- | ----------------------------------- | ---------------------- |
| Concurrent    | 1 MMS, 2 SB      | SB1 + SB2 simultaneous              | One fails              |
| Sequential    | 1 MMS, 2 SB      | SB1, wait, SB2                      | Both OK                |
| Two MMS       | 1 MMS, 2 SB      | First concurrent, second concurrent | First fails, second OK |
| Dual MMS      | 2 MMS, 1 SB each | MMS1.SB + MMS2.SB simultaneous      | One fails              |
| Single Append | 2 MMS, 1 SB each | Only MMS1 appends                   | OK                     |

### What We Observe

- `mediaparserd` connection activates during the append operations
- `mediaparserd` logs error `-19431` in `FigApplicationStateMonitor` (but this happens even in successful cases)
- One of two concurrent parsing requests returns `ParsingFailed`
- The failure is non-deterministic

### What We Can't Prove

1. **Where exactly the bug is** - could be mediaparserd, GPU process, WebKit IPC, or elsewhere
2. **Whether this is a WebKit bug or iOS/CoreMedia bug** - we only see what the logs show
3. The exact race condition mechanism
4. The meaning of error code `-19431`

### Conclusion

Concurrent `appendBuffer` calls during cold start trigger a race condition at some shared layer. One of the concurrent requests will fail with `ParsingFailed`. Creating MMS/SourceBuffers is fine - only concurrent appends trigger the issue. We observe mediaparserd in the logs but cannot definitively identify the exact location of the bug.
