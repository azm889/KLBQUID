# KLBQUID
通过中继链接游戏大厅的卡拉彼丘UID查询

## 1. 中继的目的
游戏客户端登录后，会一直连着「大厅服务器」（TCP 长连接，protobuf 帧）。  
中继把自己插在中间：
游戏客户端──> 中继(10801) ──TCP──> 大厅服务器
                  │
                  ├─ 记下这条已登录的连接
                  ├─ 需要查询时，注入一条请求帧
                  └─ 截获响应帧 → 解析 → 通过 HTTP/文件给出去

## 2. 连接游戏大厅
目标：让游戏访问 `klbqcp-prod-dir.gxpan.cn` 的流量先到本机 `10801`。
原理：
游戏会连这个域名的两个端口：
- `11001`：登录端口
- `14011 / 14012`：大厅端口
只要把这个域名指到中继的 SOCKS5 端口即可。

## 3. 怎么发请求

### 3.1 HTTP 接口
所有 POST 都要带：

```
Authorization: Bearer <--http-token 的令牌>
Content-Type: application/json
```

| 方法 | 路径 | 请求体 | 用途 |
| --- | --- | --- | --- |
| GET | `/health` | 无（免鉴权） | 健康检查 |
| POST | `/query` | `{"nickname":"压力思冯"}` 或 `{"uid":"12865235"}` | 玩家资料 |
| POST | `/matches` | `{"uid":"50559443","page":1}` 或 `{"nickname":"..."}` | 最近对局 |
| POST | `/season` | `{"uid":"50559443","season":17}` | 指定赛季数据 |
| POST | `/leaderboard` | `{"bodies":[5 个 base64]}` | 国服前 100 |

### 3.2 示例

```bash
# 健康
curl http://127.0.0.1:10802/health

# 按昵称查资料
curl -X POST http://127.0.0.1:10802/query \
  -H "Authorization: Bearer my-token-123" \
  -H "Content-Type: application/json" \
  -d '{"nickname":"压力思冯"}'

# 按 UID 查资料
curl -X POST http://127.0.0.1:10802/query \
  -H "Authorization: Bearer my-token-123" \
  -H "Content-Type: application/json" \
  -d '{"uid":"12865235"}'

# 最近对局
curl -X POST http://127.0.0.1:10802/matches \
  -H "Authorization: Bearer my-token-123" \
  -H "Content-Type: application/json" \
  -d '{"uid":"50559443","page":1}'

# 赛季生涯资料
curl -X POST http://127.0.0.1:10802/season \
  -H "Authorization: Bearer my-token-123" \
  -H "Content-Type: application/json" \
  -d '{"uid":"50559443","season":17}'
```

### 3.3 文件模式

请求写到 `KLBQ_RELAY_STATE\klbq-query-request.json`：

```json
{"nickname": "压力思冯"}
{"uid": "12865235"}
{"type": "matches", "uid": "50559443", "page": 1}
{"type": "season", "uid": "50559443", "season": 17}
{"type": "seasons", "uid": "50559443", "last": 26}
{"type": "leaderboard", "bodies": ["<base64>", "..."]}
```

中继把结果写到 `klbq-query-result.json`，结构同 HTTP 返回值。

### 3.4 批量赛季

`/season` 一次一个赛季；要批量用文件模式的 `seasons`：

```json
{"type": "seasons", "uid": "50559443", "last": 26}
```

中继会从 1 查到 `last`，每个赛季间隔 2 秒，遇到限流自动重试，最后返回：

```json
{"player_id": "50559443", "last": 26, "seasons": [{"season": 17, "modes": [...], "roles": [...]}]}
```

没有数据的赛季（没建号/0 场）不会出现在 `seasons` 里。

---

## 4. 返回数据怎么解析

### 4.1 先理解帧

```
0..1  整帧长度
2     方向
3     序号
4..5  消息号
6..11 固定 0
12..  protobuf 正文
```

中继的 HTTP/文件接口已经帮你拆掉了帧头，返回的是**正文解析结果**；只有 `career_body_b64`、`search_body_b64`、`pages` 是原始正文（base64），需要自己再解。

### 4.2 通用字段读取器

`klbq_inject.py` 里已有 `_fields_all`，支持 protobuf 的 wire 0/1/2/5：

```python
import struct
from klbq_inject import _fields_all

def read(blob):
    """{字段号: 值}，浮点自动解开。"""
    out = {}
    for number, wire, value in _fields_all(blob):
        if wire == 0:
            out[number] = value
        elif wire == 5:
            out[number] = struct.unpack("<f", value)[0]
        elif wire == 1:
            out[number] = struct.unpack("<d", value)[0]
        elif wire == 2:
            out.setdefault("bytes", {})[number] = value
    return out
```

### 4.3 `/query` 的 `players`（11112 的 player）

`players` 里每项就是 `ncmd_cs.player` 的部分字段：

| 字段 | 含义 |
| --- | --- |
| 1 | player_id（UID） |
| 2 | 昵称 |
| 3 | 基板 id |
| 5 | 段位 id |
| 10 | 在线状态枚举 |
| 12 | 胜场 |
| 13 | MVP 次数 |
| 14 | 总场次 |
| 15 | 场均击杀（×100） |
| 17 | 等级 |
| 23 | 最近离线时间戳 |
| 27 | 平台 |
| 29 | 边框 id |
| 34 | 段位名次 |
| 35 | 段位分 |
| 37 | 挑战赛分数 |
| 38 | 城市代码 |
| 16 | 常用角色（packed） |
| 52 | 晶卡 |
| 55 | 各模式段位 `{1: type, 2: division, 3: score}` |

### 4.4 生涯：`career_body_b64`（6801）

结构：

```
{1: code, 2: career}

career.1  = 身份块（1=UID，2=昵称）
career.2  = 重复项：按模式统计
career.5  = 重复项：模式 × 角色
career.4  = 收藏（皮肤数量）
career.6 / 7 / 11 / 33 / 35 / 36 = 段位相关
```

解析代码：

```python
import base64
from klbq_inject import _fields_all

def parse_career(body):
    top = {n: v for n, w, v in _fields_all(body) if w == 0}
    if top.get(1, 0) != 0:
        raise ValueError("code != 0")
    career = next(v for n, w, v in _fields_all(body) if n == 2 and w == 2)
    cf = list(_fields_all(career))
    ident = next(v for n, w, v in cf if n == 1 and w == 2)
    ident_fields = {n: v for n, w, v in _fields_all(ident) if w == 0}

    def row(blob):
        out = {}
        for n, w, v in _fields_all(blob):
            if w == 0:
                out[n] = v
            elif w == 5:
                out[n] = struct.unpack("<f", v)[0]
        return out

    modes = [row(v) for n, w, v in cf if n == 2 and w == 2]
    roles = [row(v) for n, w, v in cf if n == 5 and w == 2]
    scalars = {n: v for n, w, v in cf if w == 0}

    return {
        "uid": str(ident_fields.get(1)),
        "nick": ident_fields.get(2, b"").decode("utf-8", "ignore"),
        "modes": [
            {
                "mode": r.get(1),
                "games": r.get(2),
                "kills": r.get(15),
                "deaths": r.get(24),
                "wins": r.get(27),
                "avg_damage": r.get(37),
                "kd": r.get(45),
                "headshot": r.get(46),
                "win_rate": r.get(52),
            }
            for r in modes
        ],
        "roles": [
            {
                "mode": r.get(1),
                "role_id": r.get(2),
                "games": r.get(3),
                "kills": r.get(8),
                "deaths": r.get(9),
                "best_kills": r.get(12),
                "best_damage": r.get(13),
                "win_rate": r.get(16),
                "score": r.get(18),
            }
            for r in roles
        ],
        "rank": {
            "season_high": scalars.get(6),
            "current": scalars.get(7),
            "pos": scalars.get(11),
            "history_high": scalars.get(33),
            "history_pos": scalars.get(35),
            "history_season": scalars.get(36),
        },
    }
```

字段说明：

- 模式行：`1=mode, 2=场次, 15=击杀, 24=死亡, 27=胜场, 37=回合均伤, 45=K/D, 46=爆头率, 52=胜率`
- 角色行：`1=mode, 2=角色 id, 3=场次, 8=击杀, 9=死亡, 12=单局最高击杀, 13=单局最高伤害, 16=胜率, 18=表现分`
- 段位：`6=本赛季最高, 7=当前, 11=当前名次, 33=历史最高, 35=历史名次, 36=历史赛季`
- 收藏（career.4）：`7=角色皮肤拥有, 8=角色皮肤总数, 3=武器皮肤拥有, 4=武器皮肤总数`

模式 id：

| id | 模式 | id | 模式 |
| --- | --- | --- | --- |
| 2 | 一般爆破 | 13 | 晶源感染 |
| 5 | 排位爆破 | 18 | 大头乱斗 |
| 9 | 极限推进 | 28 | 极限刀战 |
| 30 | 枪王乱斗 | 35 | 晶源追击 |

### 4.5 对局列表：`/matches`（7101）

返回已经是整理好的 `matches` 数组，每项包含 `room_id / mode / play_mode / map_id / played_at / duration / result_raw / raw`。

真正要用的字段在 `raw` 里（键是字符串，先转 int）：

| raw 键 | 含义 |
| --- | --- |
| 1 | 房间模式（**5=排位爆破**） |
| 2 | 玩法（2=爆破，1=团竞） |
| 3 | 地图 id |
| 7 | 开始时间（Unix 秒） |
| 8 | roomId |
| 9 | 时长（秒） |
| 14 | **赛后段位分** |
| 15 | **赛前段位分** |

胜负与分差：

```python
def summarize(payload):
    rows = []
    for item in payload["matches"]:
        raw = {int(k): v for k, v in item["raw"].items()}
        before, after = raw.get(15), raw.get(14)
        delta = (after or 0) - (before or 0)
        rows.append({
            "room_id": item["room_id"],
            "mode": raw.get(1),          # 5 = 排位爆破
            "play_mode": raw.get(2),
            "map_id": item["map_id"],
            "duration": item["duration"],
            "played_at": item["played_at"],
            "win": delta > 0,
            "delta": delta,
        })
    return rows
```

> 不要用 `result_raw`（字段 5）判断胜负，它在不同对局里不稳定；用 `f14 - f15` 的正负，已逐场核对。

地图 id：

| id | 名称 | id | 名称 |
| --- | --- | --- | --- |
| 200 | 404基地 | 205 | 柯西街区 |
| 201 | 88区 | 206 | 科斯迷特 |
| 202 | 欧拉港口 | 207 | 奥卡努斯 |
| 203 | 风曳镇 | 208 | 莱布伦城 |
| 204 | 空间实验室 | | |

### 4.6 赛季数据：`/season`（6809）
返回：

```json
{"season": 17, "code": 0, "modes": [...], "roles": [...], "fields": {...}}
```

`fields` 的键是字符串，先转 int：

```python
def parse_season(payload):
    fields = {int(k): v for k, v in payload["fields"].items()}
    modes = [{int(k): v for k, v in r.items()} for r in payload["modes"]]
    roles = [{int(k): v for k, v in r.items()} for r in payload["roles"]]
    ranked = next((r for r in modes if r.get(1) == 5), {})
    return {
        "season": payload.get("season"),
        "rank": fields.get(3),
        "rank_high": fields.get(4),
        "score": fields.get(15),
        "games": ranked.get(2),
        "wins": ranked.get(27),
        "kills": ranked.get(15),
        "deaths": ranked.get(24),
        "assists": ranked.get(17),
        "plants": ranked.get(20),
        "defuses": ranked.get(21),
        "skill_kills": ranked.get(22),
        "melee_kills": ranked.get(43),
        "roles": [
            {
                "role_id": r.get(2),
                "games": r.get(3),
                "wins": r.get(4),
                "kills": r.get(8),
                "deaths": r.get(9),
                "best_kills": r.get(12),
                "score": r.get(18),
            }
            for r in roles if r.get(1) == 5
        ],
    }
```

限流响应长这样：

```json
{"code": 8610, "season": 16, "fields": {...}}
```

看到 `code == 8610` 就等 1~2 秒重试。

### 4.7 排行榜：`/leaderboard`（11402）

返回 5 页 base64（`pages`），每页正文用 `parse_rank_page` 解析：

```python
from account_protocol import parse_rank_page

for page in payload["pages"]:
    metadata, rows = parse_rank_page(base64.b64decode(page))
    # rows: position / player_id / nickname / level
    #       rank_raw / division_score_raw / grade_score_raw
```

校验：5 页共 100 人，名次 1..100 连续。

---
### 4.3 错误码

| error | 含义 | 
| --- | --- |
| `no_lobby_session` | 中继没接上大厅 | 
| `session_closed` / `session_replaced` | 
| `timeout` | 查询超时 | 
| `rate_limited` | 服务端限流 | 
| `player_not_found` | 查无此人 | 
| `nickname_required` | 缺参数 | 
| `descriptors_unavailable` | 描述符缺失 | 
| `connection_error` | 网络问题 | 

---

## 5. 示例
<img width="1280" height="2490" alt="6f996c45ea62c77b64f6023850023507" src="https://github.com/user-attachments/assets/2c454a31-ee59-461f-a8dc-addf25b508e1" />
