# Chúng tôi đã giải bài toán C10K thế nào

> Hôm rồi có ông nhắc lại bài toán C10K trên timeline — tự dưng cả đống ký ức 15 năm trước ùa về. Hồi 2008-2009 anh em tôi cũng đập đầu vào bài này, và cuối cùng tìm ra một lời giải mà bây giờ ngẫm lại thấy khá đặc biệt: chúng tôi tình cờ "phát minh lại" async/await bằng `yield` của Python — trước cả khi keyword `async`/`await` chính thức ra đời. Tiếc là không công bố, không open source. Viết lại chuyện ngày đó cho anh em đọc.

## Bối cảnh

Năm 2008-2009 là thời web bùng nổ. Anh em làm backend hồi đó ai cũng đụng cùng một câu hỏi: làm sao cho một con server cõng được mười nghìn connection cùng lúc? Anh mẽo Dan Kegel gọi nó là C10K problem.

Cách cũ là thread-per-connection, kiểu Apache prefork. Code đọc dễ, debug dễ, nhưng chết ở scale. Mười nghìn thread tức là mất vài chục GB RAM chỉ để... ngồi đợi I/O. Cộng thêm context switch liên tục khiến scheduler ngộp thở. Resource cháy hết cho overhead chứ không cho work thật. Phải nghĩ cách khác.

## Bài học đầu tiên từ nginx

Anh Nga ngố Igor Sysoev làm nginx từ 2004 đi đường khác hẳn: event-driven, non-blocking I/O, một worker process cõng hàng vạn connection. Mổ source ra thì cái core của nginx đơn giản đến mức choáng — chỉ là một vòng lặp:

```c
for (;;) {
    timer = find_next_timer();
    events = wait_for_io(timer);   // chỗ duy nhất block
    for (each event in events) {
        handler(event);             // không bao giờ block
    }
    process_timers();
}
```

Dòng `wait_for_io()` thì tùy OS. Windows hồi đó chỉ có `select()` chạy O(n) theo số fd, còn Linux 2.6 đã có `epoll` chạy O(1), gồng được hàng trăm nghìn fd. Anh em đi Linux là chuyện đương nhiên.

Có điều codebase nginx to oành, đầy macro của anh Igor, viết để chạy production chứ không phải để học. Đọc mãi không vô. May mà đầu năm 2009 có một thứ vừa ra đời.

## Mổ source Redis để học event loop

Tháng 2/2009 anh Ý đảo Sicilia antirez công bố Redis. Một thread, một event loop, mà throughput khủng. Bí mật nằm gọn trong vài file: `ae.h` cho interface, `ae.c` cho core (chỉ tầm 400 dòng), rồi `ae_epoll.c`/`ae_kqueue.c`/`ae_select.c` cho từng OS. Tên `ae` viết tắt của "A simple Event-driven programming library". Cả file chính đọc một buổi sáng là xong.

API public gọn lỏn vài hàm: tạo/hủy event loop, đăng ký file event cho một fd, time event, `aeMain` để khởi động vòng lặp. Phần platform-specific giấu sạch sau wrapper thống nhất. Anh em tôi bê nguyên `ae` về xài luôn, không động dao vào bên trong. Cái transfer được sang tầng trên framework là *kiến trúc* — cách tách public API khỏi platform-specific code — chứ không phải code.

## Event-driven dễ như ăn kẹo

Có cái event loop từ Redis trong tay rồi, viết event-driven dễ như ăn kẹo. Có fd ready? Đăng ký callback. Cần timer? Đẩy vào time event queue. Engine `ae` lo nốt.

Cái HTTP server tự nhiên rơi xuống — accept-loop trên port 80, parse request ra, tra routing table, gọi handler đã đăng ký:

```python
@http.get("/users/*")
def get_user(httpContext):
    user_id = httpContext.path.split("/")[-1]
    # ... build response ...
```

Cái pattern đó áp dụng cho mọi protocol — không cứ HTTP. Hồi anh em làm game backend, chỉ cần thay parser và dispatcher, đăng ký handler theo opcode thay vì path:

```python
@socket_event(opcode=0x0A)        # MOVE
def on_move(player, dx, dy):
    player.x += dx; player.y += dy
    broadcast_to_room(player.room_id, MoveEvent(player.id, dx, dy))
```

Engine không đổi một dòng. Cùng một backend vừa cõng REST API cho team product, vừa cõng game realtime cho team game. Đây mới đúng là cái mà anh em hay gọi là "platform" — không phải một con server giải C10K, mà một cái nền cho nhiều sản phẩm khác nhau đứng lên.

## Nhưng I/O đi ra mới là nửa còn lại

Đến đây Redis giải đẹp **một nửa** bài toán: phần *connection đi vào*. Còn **nửa kia** — phần *I/O đi ra* — thì Redis bó tay. Vì khi handler chạy, nghiệp vụ thật mới bắt đầu: query MySQL lấy user, gọi HTTP service nội bộ lấy profile, ghi log, query tiếp rồi mới response. Mọi bước I/O đó phải non-blocking, vì chỉ cần một query MySQL chậm là đóng băng cả 9.999 connection còn lại đang share event loop.

Hồi 2008-2009 anh em mới chỉ có hai lựa chọn để viết code non-blocking, cả hai đều khó chịu.

**Thread-per-connection** thì đọc dễ — tuần tự, dễ debug, dễ try/catch — nhưng chết với 10K, đốt cả đống thread chỉ để ngồi đợi I/O đi ra.

**Event-driven với callback** là cách Node.js (vừa ra 2009) đang đẩy lên trend. Nhìn lại đoạn code dưới đây thì khó chịu thí mịa, nhưng hồi Node.js vừa ra anh em cũng phát cuồng — cảm giác mới mẻ, hiện đại, "đây mới là tương lai", chưa ai gọi nó là callback hell. Phải đến lúc viết app thật, lồng năm bảy tầng cho một flow nghiệp vụ phức tạp, mới thấy đau. Cùng flow trên viết kiểu callback thành thế này:

```python
def handle_request(conn):
    def on_read(req):
        def on_user(user):
            def on_profile(profile):
                conn.write(render(user, profile))
            http.get_async(f"http://profile-svc/{user.id}", on_profile)
        db.query_async("SELECT * FROM users WHERE id=?", req.user_id, on_user)
    conn.read_async(on_read)
```

Bốn cấp indent cho một flow tầm thường. Try/catch vô nghĩa vì exception ném từ callback có lan ngược lên đâu. Stack trace cụt ngủn ở event loop.

Workaround của anh em hồi đó là tống logic xuống stored procedure MySQL — một `CALL sp_handle_order(...)` thay cho năm callback lồng nhau. Sạch trên giấy, đau khổ trong dài hạn. Logic chia đôi application với SQL. Deploy đồng bộ là cơn ác mộng. Database thành chỗ chứa business logic. Đau nhất là code không test được — phải dựng MySQL thật, seed data thật, gọi `CALL` thật. Đau thứ hai là SQL không làm được nhiều hàm cần thiết, anh em phải chế hẳn MySQL plugin viết bằng C, compile thành shared library load vào MySQL server. Đến lúc phải hack MySQL bằng C plugin chỉ để vá một mô hình lập trình sai từ đầu thì rõ ràng là đã đi quá xa rồi.

## Khoảnh khắc gặp `yield`

Một hôm lang thang đọc tài liệu Python 2.5/2.6, tôi vớ phải một feature thực ra đã có từ PEP 255 năm 2001: generator với keyword `yield`. Cái cần nói trước: Python thêm `yield` *không phải* để giải concurrency. Nó sinh ra để viết iterator cho gọn — thay vì phải làm cả một class với `__iter__`/`__next__` chỉ để sinh dãy số:

```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b
```

Đến PEP 342 năm 2005 thì Python mở rộng thêm `.send()` để generator nhận giá trị truyền ngược vào — nhưng vẫn được coi là "iterator nâng cao". Đây là feature ngôn ngữ làm sẵn cho mục đích khác, đang chờ ai đó nhìn ra cách dùng thứ hai.

Cái đặc tính khiến `yield` thành chìa khóa lại là một thuộc tính phụ của cơ chế iterator: **một hàm có thể dừng giữa chừng, trả quyền lại cho ngoài, rồi được caller "đánh thức" để chạy tiếp từ chính chỗ vừa dừng** — với toàn bộ biến local vẫn nguyên xi.

Quan trọng hơn nữa là `yield` là một biểu thức **hai chiều**: vừa đẩy giá trị ra ngoài cho caller, vừa nhận giá trị từ caller chui ngược vào trong khi resume (qua `.send()`). Cần hai cái yield mới thấy rõ luồng:

```python
def gen():
    x = yield 10        # khi resume, send-value chui vào đây thành x
    y = yield 20        # lần resume kế tiếp, send-value chui vào đây thành y
    return x + y
```

```
caller                          generator

g = gen()                       (paused, chưa chạy dòng nào)

val = next(g)         ──►       chạy đến `yield 10`, dừng, đẩy 10 ra
val == 10             ◄──

val = g.send(100)     ──►       100 chui vào, x = 100
                                chạy đến `yield 20`, dừng, đẩy 20 ra
val == 20             ◄──

val = g.send(200)     ──►       200 chui vào, y = 200
                                chạy đến `return x + y`
StopIteration(300)    ◄──
```

Ba điểm cần nắm. Một, `g.send(value)` đẩy `value` vào ngay chỗ `yield` đã ngủ từ lần trước — `value` thành kết quả của biểu thức `yield` đó, gán vào vế trái (`x`, `y`). Hai, generator chạy tiếp đến `yield` kế tiếp, đẩy vế phải của nó (`10`, `20`) ra. Ba, cái vế phải đẩy ra ấy chính là return value của `send`. Cặp `(x, y)` là input từ caller chui vào; cặp `(10, 20)` là output đi ra. Hai chiều.

Có cơ chế này thì runner mới làm được việc của nó: ném kết quả I/O vừa xong vào generator (qua `send`), nhận `YieldReturn` kế tiếp ra (qua return value của `send`), lặp lại. Còn ai sẽ gọi `.send()` để đánh thức generator khi I/O xong? Câu trả lời lộ rõ luôn: chính cái event loop từ Redis `ae.c`.

## Ráp lại: coroutine model

Ý tưởng rất gọn. Khi code application cần I/O, nó *không* gọi callback — nó `yield` ra một object mô tả việc cần làm. Bên ngoài có một function `run` đang drive generator: nhận object về từ `gen.send`, gọi method polymorphic `schedule()` của nó, nhận kết quả, ném ngược vào gen, lặp lại. Khi gen xong (`StopIteration`), trả về final value.

Anh em tôi gọi mỗi hàm như vậy là một **task** — cái tên đơn giản nhất nghĩ ra được lúc đó. (Mãi sau cái từ học thuật *coroutine* mới thành chuẩn chung.) Cùng flow nghiệp vụ ở trên, viết bằng task với `yield`:

```python
@task
def handle_request(conn):
    req     = yield conn.read()
    user    = yield db.query("SELECT * FROM users WHERE id=?", req.user_id)
    profile = yield http.get(f"http://profile-svc/{user.id}")
    yield log.write(f"served {user.id}")
    yield conn.write(render(user, profile))
```

Cùng logic, cùng performance, chạy trên cùng cái event loop một thread cõng hàng vạn connection. Khác mỗi mấy chữ `yield` rải đúng chỗ.

Toàn bộ pattern gói trong vài chục dòng. Mỗi loại I/O là một subclass nhỏ của `YieldReturn`, mỗi cái override `schedule()` riêng — không có `isinstance` dispatch ở chỗ nào hết:

```python
import functools

class YieldReturn:
    def schedule(self):
        raise NotImplementedError

class MySqlReturn(YieldReturn):
    def __init__(self, sql): self.sql = sql
    def schedule(self):
        return loop.await_mysql(self.sql)        # bridging callback ↔ event loop giấu trong primitive

class ReadReturn(YieldReturn):
    def __init__(self, fd, nbytes=4096): self.fd, self.nbytes = fd, nbytes
    def schedule(self):
        return loop.read_when_ready(self.fd, self.nbytes)

class db:
    @staticmethod
    def query(sql): return MySqlReturn(sql)

def run(gen):
    """Drive gen đến hết, trả về final value."""
    value = None
    while True:
        try:
            yr = gen.send(value)
        except StopIteration as stop:
            return getattr(stop, 'value', None)
        value = yr.schedule()

class TaskReturn(YieldReturn):
    """YieldReturn bọc một gen. Triết lí: yield một YieldReturn = chạy đến hết."""
    def __init__(self, gen): self.gen = gen
    def schedule(self):
        return run(self.gen)

def task(genfunc):
    @functools.wraps(genfunc)
    def wrapper(*args, **kwargs):
        return TaskReturn(genfunc(*args, **kwargs))
    return wrapper
```

Mental model thống nhất: *gọi để tạo, yield để chạy và lấy kết quả*. `db.query(sql)` tạo ra `MySqlReturn`, `yield db.query(sql)` chạy query và trả result. `get_user_full(id)` tạo ra `TaskReturn`, `yield get_user_full(id)` chạy sub-task đến hết rồi trả final value. Compose sub-task tự nhiên, không cần `yield from`.

Cần compose phức tạp hơn thì thêm subclass mới. `YieldParallel(*tasks)` chạy song song nhiều task rồi đợi tất cả về — cái mà JavaScript sau này gọi là `Promise.all`, Python gọi là `asyncio.gather`. `YieldFirst(*tasks)` chạy song song nhưng lấy cái về trước tiên, dùng cho timeout/fallback/query nhiều replica — `Promise.race`, `asyncio.wait(FIRST_COMPLETED)`. Mỗi cái dưới ba mươi dòng, không động đến core.

## Sau khi core đúng, mọi thứ tự đến

Cái khó dồn hết vào việc giải đúng core — `YieldReturn` + `run` + decorator. Xong cái đó rồi thì mọi nhu cầu sau này đều đến tự nhiên, lời giải cũng tự lộ ra.

Connection pooling: thêm subclass `MySqlPoolReturn` — check pool có connection rỗi không, có thì query, không thì đăng ký vào hàng đợi rồi "ngủ" cho đến khi có conn trả về. Dev viết ứng dụng vẫn cứ `yield db.query(sql)` như cũ, không quan tâm bên dưới đang direct hay pool. Rate limiting thành subclass đợi token. Retry với backoff là subclass bọc sub-task rồi schedule lại với delay. Circuit breaker check state trước khi cho schedule chạy. Mỗi nhu cầu một subclass, không quá vài chục dòng. Khi có tính năng mới, anh em không hỏi "phải sửa chỗ nào trong scheduler" mà chỉ hỏi "thêm subclass nào".

Mấy cái decorator dispatch ở section đầu cũng tự được nâng cấp. Handler giờ yield I/O ngay bên trong, không cần callback nữa — `@http.get` và `@socket_event` không đổi cơ chế (vẫn routing table, vẫn accept-loop) — chỉ là body của handler giờ viết theo coroutine sequential cho dễ. Hai layer dispatch và coroutine đi độc lập, ghép vào thì cộng hưởng.

## Nhìn lại

Framework gói trong ba layer xếp chồng: event loop bê nguyên từ Redis `ae` ở dưới, `YieldReturn` + `run` cộng subclass I/O ở giữa, application code với hàm task và vài chữ `yield` ở trên. Hồi đó anh em ai cũng biết là làm được một thứ rất xịn, nhưng cũng vì xịn quá nên giấu như mèo giấu cứt — không public, không viết blog, không open source. Giờ ngồi nhìn lại, có lẽ đó là cái sai lớn nhất của cả hành trình.

Vài năm sau, cả ngành tự đi đến đúng cái mô hình ấy bằng đường vòng. Tornado (2009) cũng đi lối yield-based coroutine. PEP 380 (2012) thêm `yield from`. PEP 492 (2015) thêm `async`/`await` — bản chất là syntactic sugar cho đúng cái cơ chế mà anh em đã làm bằng tay từ sáu năm trước. JavaScript theo sau với ES2017. C# có từ 5.0. Rust ổn định 2019. Nếu hồi 2009-2010 mà open source ra, có khi thế giới lập trình đã đi theo hướng async-by-default sớm hơn cả gần thập kỷ. Có khi thôi, ai mà biết được. Nhưng cái cảm giác "lẽ ra đã có thể" thì nó vẫn ở đó.

Hôm rồi đọc lại bài toán C10K mà ngẫm vẫn thấy lạ. Cái primitive đẹp nhất của câu chuyện này — `yield` — nó nằm sẵn đó trong Python từ 2001, sờ sờ ra đấy cho cả thế giới. Anh em tôi tình cờ thấy nó vào đúng lúc đang đau với callback hell, ráp được vào event loop của Redis, ra một thứ mà cả ngành phải mất thêm gần một thập kỷ mới chính thức hóa lại. Và rồi giấu kỹ trong nhà công ty, không nói với ai. Viết bài này 15 năm sau cũng là cố trả nốt cái nợ chia sẻ ngày đó — dù bây giờ thì cả thế giới đã có `async`/`await` rồi, chẳng còn ai cần đến nữa.
