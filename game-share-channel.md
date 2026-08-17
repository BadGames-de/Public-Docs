# BadGames — Game State Share Channel

Server → client plugin message channel that reports what the minigame server is doing: which
map is loaded, when a round starts and ends, and what the lobby voted for.

- **Channel:** `badgames:game_share`
- **Direction:** server → client only. Anything a client sends on this channel is ignored.
- **Scope:** sent per player, not broadcast. Every recipient gets their own copy.

The proxy forwards this channel untouched, so a client only has to decode the payload. The
server sends unconditionally — the client does not need to announce the channel first.

---

## Frame layout

Every message is one custom payload: a 4-byte type tag followed by a length-prefixed JSON
string.

| Order | Field | Encoding |
|---|---|---|
| 1 | Type tag | `int32`, big-endian, signed |
| 2 | JSON length | `uint16`, big-endian |
| 3 | JSON body | *Java modified UTF-8*, exactly the number of bytes given above |

```
+-------------------+--------------+-------------------------+
| type tag (4 B BE) | len (2 B BE) | JSON, `len` bytes       |
+-------------------+--------------+-------------------------+
```

A real `END` frame is 37 bytes:

```
00 00 00 01   00 1F   7B 22 74 79 70 65 22 3A 31 ...
└─ tag = 1 ─┘ └len=31┘ └─ {"type":1, ...        ─┘
```

> **The string is Java "modified UTF-8", not plain UTF-8, and the length prefix is a plain
> big-endian `uint16` — not a VarInt.** Two encoding differences matter: `U+0000` is written as
> the two bytes `C0 80`, and characters outside the BMP are written as a surrogate pair of
> three-byte sequences (CESU-8) rather than one four-byte sequence. If your language has a
> `DataInput.readUTF` equivalent, use it and both details are handled for you. Decoding as
> standard UTF-8 will work for ASCII map names and then break on the first unusual character.
>
> In particular, **Minecraft's own string reader is not compatible here** — it expects a
> VarInt length prefix and standard UTF-8. See the Fabric example below.

## Type tags

| Tag | Type | Sent when |
|---|---|---|
| `0` | `START` | A round begins |
| `1` | `END` | A round finishes |
| `2` | `MAP_CHANGE` | The selected map changes while in the lobby |
| `3` | `INFO` | Immediately after the player joins the server |

Tags are positional. **New types will be appended with higher tags; existing tags will not be
renumbered.** Ignore tags you do not recognise instead of treating them as an error.

---

## Envelope

The JSON body is always the same envelope. The outer `type` is an **integer** and repeats the
frame's type tag — you can dispatch on either, and disagreement between them means a corrupt
read.

```json
{
  "type": 0,
  "data": { }
}
```

The shape of `data` depends on the type.

### `0` — START

```json
{"type":0,"data":{"mapName":"Aurora","voteResults":{"chests":"op","combat":"old"}}}
```

| Field | Type | Notes |
|---|---|---|
| `mapName` | string | Display name of the map the round is played on |
| `voteResults` | object (string → string) | Winning option per vote category |

`voteResults` keys are vote category ids and the values are the winning option id within that
category. **Both sets are configuration-driven and vary per gamemode** — treat this as an open
map, not a fixed set of keys. The example above is illustrative only. If a server has nothing
to vote on, the object is present but empty:

```json
{"type":0,"data":{"mapName":"Aurora","voteResults":{}}}
```

### `1` — END

```json
{"type":1,"data":{"type":"END"}}
```

Carries no payload of its own.

> **Watch the two different `type` fields.** The outer one is the integer tag (`1`). The one
> inside `data` is the type *name* as a string (`"END"`), and it exists **only** on this
> message. A decoder that reads `type` from the wrong nesting level will get a string where it
> expected an int, or nothing at all. Always dispatch on the outer tag.

### `2` — MAP_CHANGE

```json
{"type":2,"data":{"newMapName":"Aurora","oldMapName":"Canyon"}}
```

| Field | Type | Notes |
|---|---|---|
| `newMapName` | string | Map now selected |
| `oldMapName` | string | Map that was selected before |

Fired while the lobby is still open, so it can arrive several times before `START`, and the map
named by `START` is the authoritative one.

### `3` — INFO

```json
{"type":3,"data":{"serverName":"sw-1","currentMap":"Aurora"}}
```

| Field | Type | Notes |
|---|---|---|
| `serverName` | string | Identifier of the server instance the player is on |
| `currentMap` | string | Map selected at the moment the player joined |

This is the only message sent on join, and it is the way to learn the initial state.

---

## Things to keep in mind

**Do not use `data.type` for dispatch.** Only `END` carries it, and it is a string while the
outer `type` is an int. Dispatch on the frame's type tag or the outer `type`.

**Any field may be missing.** Null-valued fields are dropped during serialisation rather than
emitted as `null`. Decode every field as optional and have a fallback, especially if a server
is running an older build than the one this document describes.

**Ignore unknown fields.** Fields will be added to existing payloads over time. A strict
decoder that rejects unknown keys will break on the next release.

**Joining mid-round does not replay `START`.** A player who connects while a round is running
receives `INFO` only. If you need vote results for a round already in progress, you cannot get
them from this channel — `INFO` gives you the map, not the vote outcome.

**`START` is sent to whoever is online at that moment.** Anyone connecting later misses it.

**Register your receiver at mod init, not lazily.** `INFO` is sent as part of the join flow,
and an unregistered payload type is discarded before your code ever sees it.

**Payloads are small by design.** The length prefix caps the JSON at 65,535 bytes, and the
transport imposes a lower practical limit. Do not assume large payloads will ever arrive
intact; if a future type needs bulk data it will be framed differently.

**No request/response.** The server pushes state; there is no way to ask for it. If you miss a
message, wait for the next state change.

**Messages are not ordered against unrelated game events.** Delivery is per player and
asynchronous with respect to anything else the client observes (chat, titles, world changes),
so do not rely on a message arriving before or after a visible in-game effect.

---

## Fabric example

For Minecraft 1.21+ / Fabric API networking v2. Yarn mappings.

The payload is kept as raw bytes and parsed in the handler, because the frame is not made of
Minecraft's own primitives — see the encoding warning above.

**1. Payload type**

```java
public record GameSharePayload(byte[] raw) implements CustomPayload {

    public static final CustomPayload.Id<GameSharePayload> ID =
            new CustomPayload.Id<>(Identifier.of("badgames", "game_share"));

    public static final PacketCodec<PacketByteBuf, GameSharePayload> CODEC =
            CustomPayload.codecOf(GameSharePayload::write, GameSharePayload::new);

    private GameSharePayload(PacketByteBuf buf) {
        this(readRemaining(buf));
    }

    private static byte[] readRemaining(PacketByteBuf buf) {
        byte[] bytes = new byte[buf.readableBytes()];
        buf.readBytes(bytes);
        return bytes;
    }

    private void write(PacketByteBuf buf) {
        buf.writeBytes(raw);
    }

    @Override
    public CustomPayload.Id<? extends CustomPayload> getId() {
        return ID;
    }
}
```

**2. Register and receive** (client mod initialiser)

```java
PayloadTypeRegistry.playS2C().register(GameSharePayload.ID, GameSharePayload.CODEC);

ClientPlayNetworking.registerGlobalReceiver(GameSharePayload.ID, (payload, context) -> {
    try (DataInputStream in = new DataInputStream(new ByteArrayInputStream(payload.raw()))) {
        int tag = in.readInt();
        String json = in.readUTF();   // consumes the uint16 length + modified UTF-8 for you

        JsonObject envelope = JsonParser.parseString(json).getAsJsonObject();
        JsonObject data = envelope.getAsJsonObject("data");

        switch (tag) {
            case 0 -> onStart(data.get("mapName").getAsString(), data.getAsJsonObject("voteResults"));
            case 1 -> onEnd();
            case 2 -> onMapChange(data.get("newMapName").getAsString());
            case 3 -> onInfo(data.get("serverName").getAsString(), data.get("currentMap").getAsString());
            default -> { }   // unknown tag: ignore, do not throw
        }
    } catch (Exception exception) {
        LOGGER.warn("Malformed badgames:game_share payload", exception);
    }
});
```

**Notes on this example**

- `in.readUTF()` is the counterpart to how the string is written. **Do not use
  `buf.readString()`** — that reads a VarInt-prefixed standard-UTF-8 string and will
  mis-parse this frame.
- `readInt()` on `DataInputStream` is big-endian, which matches the tag. Note that
  `PacketByteBuf.readInt()` is also big-endian, so reading the tag straight off the buf works
  too — it is only the *string* that is incompatible.
- Fabric dispatches play-channel receivers on the client thread, so you can touch game state
  directly. If you are unsure, `context.client().execute(...)` is always safe.
- Wrap parsing in a `try`/`catch` and log rather than throw. A malformed or future-shaped
  payload should not take down your receiver.
- Guard the `data.get(...)` calls if you want to be robust against missing fields, per the
  optional-field note above; they are unguarded here only to keep the example readable.
