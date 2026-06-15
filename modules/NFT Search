# meta developer: @puremodules
# meta description: Поиск NFT-подарков Telegram по NFT, виду, цвету и цене.

import asyncio
import contextlib
import math
import time
from typing import Dict, List, Optional

from herokutl.tl import functions, types
from herokutl.tl.types import Message

from .. import loader, utils
from ..inline.types import InlineCall


def _tg(doc_id: int, fallback: str) -> str:
    return f'<tg-emoji emoji-id="{doc_id}">{fallback}</tg-emoji>'


def _btn(text: str, emoji_id: int, **kwargs) -> dict:
    button = {"text": text, "emoji_id": str(emoji_id)}
    button.update(kwargs)
    return button


ID_WORLD = 5445326466067754897
ID_KIND = 5447602197439218445
ID_DEVICE = 5445059250382469069
ID_COLOR = 5447642819239903347
ID_MONEY = 5444860552310457690
ID_CURRENCY = 5447579253723918909
ID_STAR = 5444939047132755535
ID_SEARCH = 5444989577422993015
ID_WARN = 5447381715293074599
ID_OK = 5444987348334965906
ID_INFO = 5247236071795754971
ID_LINK = 5447479640547428304
ID_FOLDER = 5444965220663458467
ID_NOTE = 5444889156792646660
ID_CLOSE = 5447434637880098257
ID_BACK = 5447506720316225765
ID_NEXT = 5445350109862720603
ID_RESET = 5445005936953424165
ID_REFRESH = 5445388803223091254
ID_ANY = 5445210909972655435

E_WORLD = _tg(ID_WORLD, "🌐")
E_NFT = _tg(5260681660189408650, "💎")
E_KIND = _tg(ID_KIND, "🌐")
E_COLOR = _tg(ID_COLOR, "🎨")
E_MONEY = _tg(ID_MONEY, "💰")
E_CURRENCY = _tg(ID_CURRENCY, "💲")
E_STAR = _tg(ID_STAR, "⭐️")
E_SEARCH = _tg(ID_SEARCH, "🔍")
E_WARN = _tg(ID_WARN, "⚠️")
E_OK = _tg(ID_OK, "✅")
E_INFO = _tg(ID_INFO, "ℹ️")
E_LINK = _tg(ID_LINK, "🔗")
E_FOLDER = _tg(ID_FOLDER, "📁")
E_NOTE = _tg(ID_NOTE, "📝")
E_CLOSE = _tg(ID_CLOSE, "🚪")
E_BACK = _tg(ID_BACK, "⬅️")
E_NEXT = _tg(ID_NEXT, "➡️")
E_RESET = _tg(ID_RESET, "🗑")
E_ANY = _tg(ID_ANY, "☑️")

RESULTS_PER_PAGE = 5
CATALOG_CACHE_TTL = 300
ATTR_CACHE_TTL = 900
COLOR_CACHE_TTL = 900
SESSION_TTL = 6 * 60 * 60
TYPE_PAGE_SIZE = 12
KIND_PAGE_SIZE = 12
COLOR_PAGE_SIZE = 12
RESALE_FETCH_LIMIT = 20
COLOR_DISCOVERY_LIMIT = 20
SCAN_BATCH_SIZE = 8
DISCOVERY_BATCH_SIZE = 8
MAX_PAGES_PER_GIFT = 8
MAX_STORED_RESULTS = 100


@loader.tds
class GiftFinderMod(loader.Module):
    """Поиск NFT-подарков Telegram по NFT, виду, цвету и цене."""

    strings = {"name": "NFT Search"}

    def __init__(self):
        self._catalog_cache = {"ts": 0.0, "items": []}
        self._catalog_task = None
        self._gift_attr_cache = {}
        self._color_cache = {"all": {"ts": 0.0, "items": []}, "by_gift": {}}
        self._color_tasks = {}
        self._sessions: Dict[str, dict] = {}

    async def client_ready(self):
        self._prune_sessions()
        self._prime_catalog()

    @loader.command()
    async def snft(self, message: Message):
        """Открыть поиск NFT-подарков"""
        token = utils.rand(12)
        self._sessions[token] = self._new_session()
        self._prime_catalog()
        self._prime_colors()

        form = await self.inline.form(
            "👀",
            message=message,
            reply_markup=self._filters_markup(token),
            silent=True,
        )

        if form:
            edit = getattr(form, "edit", None)
            if callable(edit):
                with contextlib.suppress(Exception):
                    await edit(
                        self._render_filters_text(token),
                        reply_markup=self._filters_markup(token),
                    )
            return

        await utils.answer(
            message,
            self._render_filters_text(token)
            + f"\n\n{E_WARN} <b>Не удалось открыть inline-форму.</b>",
        )

    def _new_session(self) -> dict:
        return {
            "updated_at": time.time(),
            "color_query": "",
            "color_mode": "exact",
            "color_picker_query": "",
            "max_price": 0,
            "currency": "any",
            "gift_id": None,
            "gift_title": "",
            "model_id": None,
            "model_name": "",
            "model_picker_query": "",
            "results": [],
            "page": 0,
            "scan_states": [],
            "scan_complete": False,
            "seen_slugs": set(),
            "merge_lock": asyncio.Lock(),
            "search_lock": asyncio.Lock(),
        }

    def _touch_session(self, token: str) -> Optional[dict]:
        self._prune_sessions()
        session = self._sessions.get(token)
        if session:
            session["updated_at"] = time.time()
        return session

    def _prune_sessions(self):
        now = time.time()
        stale = [
            token
            for token, state in self._sessions.items()
            if now - state.get("updated_at", now) > SESSION_TTL
        ]
        for token in stale:
            self._sessions.pop(token, None)

    def _session_or_alert(self, token: str) -> Optional[dict]:
        return self._touch_session(token)

    def _prime_catalog(self):
        cached = self._catalog_cache
        if cached["items"] and time.time() - cached["ts"] < CATALOG_CACHE_TTL:
            return
        if self._catalog_task and not self._catalog_task.done():
            return

        self._catalog_task = asyncio.create_task(self._load_catalog())
        self._catalog_task.add_done_callback(self._catalog_task_done)

    def _catalog_task_done(self, task: asyncio.Task):
        if task is self._catalog_task:
            self._catalog_task = None
        with contextlib.suppress(Exception):
            task.result()

    async def _ensure_catalog(self) -> List[dict]:
        now = time.time()
        cached = self._catalog_cache
        if cached["items"] and now - cached["ts"] < CATALOG_CACHE_TTL:
            return cached["items"]

        if self._catalog_task and not self._catalog_task.done():
            return await self._catalog_task

        return await self._load_catalog()

    async def _load_catalog(self) -> List[dict]:
        now = time.time()
        response = await self._client(functions.payments.GetStarGiftsRequest(hash=0))
        gifts = getattr(response, "gifts", []) or []
        items = []
        for gift in gifts:
            gift_id = int(getattr(gift, "id", 0) or 0)
            if not gift_id:
                continue

            title = (getattr(gift, "title", None) or f"gift {gift_id}").strip()
            resale_count = int(getattr(gift, "availability_resale", 0) or 0)
            items.append(
                {
                    "id": gift_id,
                    "title": title,
                    "resale_count": resale_count,
                }
            )

        items.sort(key=lambda item: (item["title"].lower(), item["id"]))
        self._catalog_cache = {"ts": now, "items": items}
        return items

    def _filters_markup(self, token: str) -> list:
        return [
            [
                _btn(
                    "цвет",
                    ID_COLOR,
                    callback=self._show_color_picker,
                    args=(token, 0),
                ),
                _btn(
                    "NFT",
                    5260681660189408650,
                    callback=self._show_type_picker,
                    args=(token, 0),
                ),
            ],
            [
                _btn(
                    "вид",
                    ID_KIND,
                    callback=self._show_model_picker,
                    args=(token, 0),
                ),
                _btn(
                    "цена",
                    ID_MONEY,
                    input="введи максимум, например 2000 или 2.5. 0 = без лимита",
                    handler=self._input_price,
                    args=(token,),
                ),
            ],
            [self._currency_button(token)],
            [
                _btn(
                    "сбросить",
                    ID_RESET,
                    style="primary",
                    callback=self._reset_filters,
                    args=(token,),
                ),
                _btn(
                    "искать",
                    ID_SEARCH,
                    style="success",
                    callback=self._run_search,
                    args=(token, 0, True),
                ),
            ],
            [_btn("закрыть", ID_CLOSE, action="close", style="danger")],
        ]

    def _currency_button(self, token: str) -> dict:
        session = self._sessions.get(token) or {}
        mode = self._normalize_currency(session.get("currency"))
        labels = {"any": "Любая", "stars": "Звезды", "ton": "TON"}
        styles = {"stars": "success", "ton": "primary"}
        kwargs = {}
        if mode in styles:
            kwargs["style"] = styles[mode]
        return _btn(
            f"Валюта: {labels[mode]}",
            ID_CURRENCY,
            callback=self._cycle_currency,
            args=(token,),
            **kwargs,
        )

    def _render_filters_text(self, token: str) -> str:
        session = self._sessions.get(token) or self._new_session()
        gift_title = utils.escape_html(session.get("gift_title") or "любой NFT")
        model_name = utils.escape_html(session.get("model_name") or "любой вид")
        color = utils.escape_html(session.get("color_query") or "любой цвет")
        max_price = session.get("max_price") or 0
        price_text = f"<code>{self._format_price(max_price)}</code>" if max_price else "без лимита"
        currency = self._currency_text(session.get("currency"))

        return (
            f"{E_WORLD} <b>Поиск NFT-подарков</b>\n\n"
            f"<blockquote>{E_NFT} <b>NFT:</b> <code>{gift_title}</code>\n"
            f"{E_KIND} <b>Вид:</b> <code>{model_name}</code>\n"
            f"{E_COLOR} <b>Цвет фона:</b> <code>{color}</code>\n"
            f"{E_MONEY} <b>Макс цена:</b> {price_text}\n"
            f"{E_CURRENCY} <b>Валюта</b>: {currency}</blockquote>"
        )

    async def _refresh_filters(self, call: InlineCall, token: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        await call.edit(
            self._render_filters_text(token),
            reply_markup=self._filters_markup(token),
        )

    async def _input_price(self, call: InlineCall, query: str, token: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        raw = (
            (query or "")
            .strip()
            .lower()
            .replace("⭐", "")
            .replace("stars", "")
            .replace("звезды", "")
            .replace("звёзды", "")
            .replace("звезд", "")
            .replace("звёзд", "")
            .replace("ton", "")
            .replace("тон", "")
            .replace(" ", "")
            .replace(",", ".")
        )
        try:
            value = float(raw)
            if value < 0 or not math.isfinite(value):
                raise ValueError
        except Exception:
            await call.answer("цена должна быть числом >= 0", show_alert=True)
            return

        session["max_price"] = int(value) if value.is_integer() else value
        self._reset_search_state(session)
        await self._refresh_filters(call, token)

    async def _cycle_currency(self, call: InlineCall, token: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        order = ("any", "stars", "ton")
        mode = self._normalize_currency(session.get("currency"))
        session["currency"] = order[(order.index(mode) + 1) % len(order)]
        self._reset_search_state(session)
        await self._refresh_filters(call, token)

    def _normalize_currency(self, value: Optional[str]) -> str:
        return value if value in {"any", "stars", "ton"} else "any"

    def _currency_text(self, value: Optional[str]) -> str:
        return {
            "any": "звезды+тон",
            "stars": "звезды",
            "ton": "тон",
        }[self._normalize_currency(value)]

    async def _reset_filters(self, call: InlineCall, token: str):
        if token not in self._sessions:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        self._sessions[token] = self._new_session()
        self._prime_colors()
        await self._refresh_filters(call, token)

    def _get_attr_cache_entry(self, gift_id: int) -> dict:
        return self._gift_attr_cache.setdefault(
            str(int(gift_id)),
            {"ts": 0.0, "backdrops": [], "models": []},
        )

    async def _ensure_upgrade_attrs(self, gift_id: int) -> dict:
        entry = self._get_attr_cache_entry(gift_id)
        if entry["ts"] and time.time() - entry["ts"] < ATTR_CACHE_TTL:
            return entry

        response = await self._client(
            functions.payments.GetStarGiftUpgradeAttributesRequest(gift_id=int(gift_id))
        )

        backdrops = []
        models = []
        seen_backdrops = set()
        seen_models = set()

        for attr in getattr(response, "attributes", []) or []:
            if isinstance(attr, types.StarGiftAttributeBackdrop):
                name = (getattr(attr, "name", None) or "").strip()
                backdrop_id = int(getattr(attr, "backdrop_id", 0) or 0)
                key = (self._normalize_text(name), backdrop_id)
                if not name or key in seen_backdrops:
                    continue
                seen_backdrops.add(key)
                backdrops.append({"name": name, "backdrop_id": backdrop_id})
                continue

            if isinstance(attr, types.StarGiftAttributeModel):
                name = (getattr(attr, "name", None) or "").strip()
                document = getattr(attr, "document", None)
                model_id = int(getattr(document, "id", 0) or 0)
                key = (self._normalize_text(name), model_id)
                if not name or not model_id or key in seen_models:
                    continue
                seen_models.add(key)
                models.append({"name": name, "model_id": model_id})

        backdrops.sort(key=lambda item: self._normalize_text(item["name"]))
        models.sort(key=lambda item: self._normalize_text(item["name"]))
        entry.update({"ts": time.time(), "backdrops": backdrops, "models": models})
        return entry

    def _get_cached_backdrops(
        self, gift_id: int, allow_stale: bool = False
    ) -> List[dict]:
        entry = self._get_attr_cache_entry(gift_id)
        items = entry.get("backdrops", [])
        if not items:
            return []
        if allow_stale or time.time() - entry["ts"] < ATTR_CACHE_TTL:
            return items
        return []

    def _get_cached_models(
        self, gift_id: int, allow_stale: bool = False
    ) -> List[dict]:
        entry = self._get_attr_cache_entry(gift_id)
        items = entry.get("models", [])
        if not items:
            return []
        if allow_stale or time.time() - entry["ts"] < ATTR_CACHE_TTL:
            return items
        return []

    async def _ensure_backdrops(self, gift_id: int) -> List[dict]:
        cached = self._get_cached_backdrops(gift_id)
        if cached:
            return cached
        return list((await self._ensure_upgrade_attrs(gift_id)).get("backdrops", []))

    async def _ensure_models(self, gift_id: int) -> List[dict]:
        cached = self._get_cached_models(gift_id)
        if cached:
            return cached
        return list((await self._ensure_upgrade_attrs(gift_id)).get("models", []))

    def _color_cache_key(self, gift_id: Optional[int]) -> str:
        return "all" if not gift_id else str(int(gift_id))

    def _get_color_cache_entry(self, gift_id: Optional[int]) -> dict:
        key = self._color_cache_key(gift_id)
        if key == "all":
            return self._color_cache["all"]
        return self._color_cache["by_gift"].setdefault(key, {"ts": 0.0, "items": []})

    def _get_cached_colors(
        self, gift_id: Optional[int], allow_stale: bool = False
    ) -> List[str]:
        entry = self._get_color_cache_entry(gift_id)
        if not entry["items"]:
            return []
        if allow_stale or time.time() - entry["ts"] < COLOR_CACHE_TTL:
            return entry["items"]
        return []

    def _prime_colors(self, gift_id: Optional[int] = None):
        key = self._color_cache_key(gift_id)
        if self._get_cached_colors(gift_id, allow_stale=True) or key in self._color_tasks:
            return

        task = asyncio.create_task(self._build_colors(gift_id))
        self._color_tasks[key] = task
        task.add_done_callback(lambda _: self._color_tasks.pop(key, None))

    async def _ensure_colors(self, gift_id: Optional[int] = None) -> List[str]:
        cached = self._get_cached_colors(gift_id)
        if cached:
            return cached

        key = self._color_cache_key(gift_id)
        task = self._color_tasks.get(key)
        if task is None:
            task = asyncio.create_task(self._build_colors(gift_id))
            self._color_tasks[key] = task
            task.add_done_callback(lambda _: self._color_tasks.pop(key, None))

        return await task

    async def _build_colors(self, gift_id: Optional[int] = None) -> List[str]:
        catalog = await self._ensure_catalog()
        if gift_id:
            items = [item for item in catalog if int(item["id"]) == int(gift_id)]
        else:
            items = [item for item in catalog if item["resale_count"] > 0] or catalog

        colors = await self._load_colors_from_attributes(items)
        if not colors:
            colors = await self._load_colors_from_resale(items)

        self._get_color_cache_entry(gift_id).update(
            {"ts": time.time(), "items": colors}
        )
        return colors

    async def _load_colors_from_attributes(self, items: List[dict]) -> List[str]:
        semaphore = asyncio.Semaphore(DISCOVERY_BATCH_SIZE)

        async def _load(item: dict) -> List[str]:
            async with semaphore:
                with contextlib.suppress(Exception):
                    return [
                        backdrop["name"]
                        for backdrop in await self._ensure_backdrops(int(item["id"]))
                    ]
            return []

        batches = await asyncio.gather(*(_load(item) for item in items))
        return self._merge_string_batches(batches)

    async def _load_colors_from_resale(self, items: List[dict]) -> List[str]:
        semaphore = asyncio.Semaphore(DISCOVERY_BATCH_SIZE)

        async def _load(item: dict) -> List[str]:
            async with semaphore:
                with contextlib.suppress(Exception):
                    response = await self._client(
                        functions.payments.GetResaleStarGiftsRequest(
                            gift_id=int(item["id"]),
                            offset="",
                            limit=COLOR_DISCOVERY_LIMIT,
                            sort_by_price=True,
                        )
                    )
                    return [
                        backdrop
                        for gift in (getattr(response, "gifts", []) or [])
                        if (backdrop := self._extract_backdrop_name(gift))
                    ]
            return []

        batches = await asyncio.gather(*(_load(item) for item in items))
        return self._merge_string_batches(batches)

    def _merge_string_batches(self, batches: List[List[str]]) -> List[str]:
        unique = {
            item.strip()
            for batch in batches
            for item in batch
            if item and item.strip()
        }
        return sorted(unique, key=self._normalize_text)

    async def _show_type_picker(self, call: InlineCall, token: str, page: int = 0):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        catalog = await self._ensure_catalog()
        items = [item for item in catalog if item["resale_count"] > 0] or catalog
        total_pages = max(1, (len(items) + TYPE_PAGE_SIZE - 1) // TYPE_PAGE_SIZE)
        page = min(max(int(page), 0), total_pages - 1)
        start = page * TYPE_PAGE_SIZE
        current = items[start : start + TYPE_PAGE_SIZE]

        rows = []
        for chunk_start in range(0, len(current), 2):
            row = []
            for item in current[chunk_start : chunk_start + 2]:
                label = item["title"]
                if len(label) > 18:
                    label = label[:15] + "..."
                marker = "[x] " if session.get("gift_id") == item["id"] else ""
                row.append(
                    _btn(
                        f"{marker}{label}",
                        5260681660189408650,
                        callback=self._set_type,
                        args=(token, item["id"]),
                    )
                )
            rows.append(row)

        nav = []
        if page > 0:
            nav.append(
                _btn(
                    "назад",
                    ID_BACK,
                    callback=self._show_type_picker,
                    args=(token, page - 1),
                    style="success",
                )
            )
        if page + 1 < total_pages:
            nav.append(
                _btn(
                    "вперёд",
                    ID_NEXT,
                    callback=self._show_type_picker,
                    args=(token, page + 1),
                    style="success",
                )
            )

        rows.append([_btn("любой NFT", ID_ANY, callback=self._clear_type, args=(token,))])
        if nav:
            rows.append(nav)
        rows.append(
            [
                _btn(
                    "к фильтрам",
                    ID_FOLDER,
                    callback=self._refresh_filters,
                    args=(token,),
                    style="primary",
                ),
                _btn("закрыть", ID_CLOSE, action="close", style="danger"),
            ]
        )

        selected = utils.escape_html(session.get("gift_title") or "любой NFT")
        await call.edit(
            f"{E_NFT} <b>Выбор NFT</b>\n\n"
            f"{E_NFT} <b>Сейчас:</b> <code>{selected}</code>\n"
            f"{E_FOLDER} <b>Страница:</b> <code>{page + 1}/{total_pages}</code>",
            reply_markup=rows,
        )

    async def _set_type(self, call: InlineCall, token: str, gift_id: int):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        catalog = await self._ensure_catalog()
        item = next((item for item in catalog if item["id"] == int(gift_id)), None)
        if not item:
            await call.answer("не удалось найти этот NFT", show_alert=True)
            return

        session["gift_id"] = item["id"]
        session["gift_title"] = item["title"]
        session["model_id"] = None
        session["model_name"] = ""
        session["model_picker_query"] = ""
        self._reset_search_state(session)
        self._prime_colors(item["id"])

        with contextlib.suppress(Exception):
            await self._ensure_upgrade_attrs(item["id"])

        models = self._get_cached_models(item["id"], allow_stale=True)
        if models:
            await self._show_model_picker(call, token, 0)
            return

        await call.answer("у этого NFT нет отдельных видов")
        await self._refresh_filters(call, token)

    async def _clear_type(self, call: InlineCall, token: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        session["gift_id"] = None
        session["gift_title"] = ""
        session["model_id"] = None
        session["model_name"] = ""
        session["model_picker_query"] = ""
        self._reset_search_state(session)
        self._prime_colors()
        await self._refresh_filters(call, token)

    async def _show_model_picker(self, call: InlineCall, token: str, page: int = 0):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        gift_id = session.get("gift_id")
        if not gift_id:
            await call.answer("сначала выбери NFT", show_alert=True)
            return

        cached_models = self._get_cached_models(int(gift_id), allow_stale=True)
        if not cached_models:
            await call.answer("получаю список видов...")

        try:
            models = cached_models or await self._ensure_models(int(gift_id))
        except Exception as e:
            await call.answer(
                f"не удалось получить виды: {utils.escape_html(str(e))[:120]}",
                show_alert=True,
            )
            return

        query = self._normalize_text(session.get("model_picker_query"))
        if query:
            filtered = [
                item for item in models if query in self._normalize_text(item["name"])
            ]
        else:
            filtered = models

        total_pages = max(1, (len(filtered) + KIND_PAGE_SIZE - 1) // KIND_PAGE_SIZE)
        page = min(max(int(page), 0), total_pages - 1)
        start = page * KIND_PAGE_SIZE
        current = filtered[start : start + KIND_PAGE_SIZE]

        rows = []
        for chunk_start in range(0, len(current), 2):
            row = []
            for item in current[chunk_start : chunk_start + 2]:
                label = item["name"]
                if len(label) > 18:
                    label = label[:15] + "..."
                marker = "[x] " if session.get("model_id") == item["model_id"] else ""
                row.append(
                    _btn(
                        f"{marker}{label}",
                        ID_KIND,
                        callback=self._set_model,
                        args=(token, item["model_id"]),
                    )
                )
            rows.append(row)

        nav = []
        if page > 0:
            nav.append(
                _btn(
                    "назад",
                    ID_BACK,
                    callback=self._show_model_picker,
                    args=(token, page - 1),
                    style="success",
                )
            )
        if page + 1 < total_pages:
            nav.append(
                _btn(
                    "вперёд",
                    ID_NEXT,
                    callback=self._show_model_picker,
                    args=(token, page + 1),
                    style="success",
                )
            )
        rows.append([_btn("любой вид", ID_ANY, callback=self._clear_model, args=(token,))])
        if nav:
            rows.append(nav)
        rows.append(
            [
                _btn(
                    "к фильтрам",
                    ID_FOLDER,
                    callback=self._refresh_filters,
                    args=(token,),
                    style="primary",
                ),
                _btn("закрыть", ID_CLOSE, action="close", style="danger"),
            ]
        )

        gift_title = utils.escape_html(session.get("gift_title") or "не выбран")
        model_name = utils.escape_html(session.get("model_name") or "любой вид")
        query_text = utils.escape_html(session.get("model_picker_query") or "без фильтра")
        count_text = f"{len(filtered)} шт." if filtered else "0"
        await call.edit(
            f"{E_KIND} <b>Выбор вида NFT</b>\n\n"
            f"{E_NFT} <b>NFT:</b> <code>{gift_title}</code>\n"
            f"{E_KIND} <b>Сейчас:</b> <code>{model_name}</code>\n"
            f"{E_SEARCH} <b>Фильтр списка:</b> <code>{query_text}</code>\n"
            f"{E_NOTE} <b>Найдено:</b> <code>{count_text}</code>\n"
            f"{E_FOLDER} <b>Страница:</b> <code>{page + 1}/{total_pages}</code>",
            reply_markup=rows,
        )

    async def _input_model_picker_query(
        self, call: InlineCall, query: str, token: str
    ):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        value = " ".join((query or "").strip().split())
        if value.lower() in {"", "0", "any", "all", "любой", "все", "всё", "-"}:
            value = ""

        session["model_picker_query"] = value
        await self._show_model_picker(call, token, 0)

    async def _set_model(self, call: InlineCall, token: str, model_id: int):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        gift_id = session.get("gift_id")
        if not gift_id:
            await call.answer("сначала выбери NFT", show_alert=True)
            return

        models = await self._ensure_models(int(gift_id))
        item = next((item for item in models if item["model_id"] == int(model_id)), None)
        if not item:
            await call.answer("не удалось найти этот вид", show_alert=True)
            return

        session["model_id"] = item["model_id"]
        session["model_name"] = item["name"]
        self._reset_search_state(session)
        await self._refresh_filters(call, token)

    async def _clear_model(self, call: InlineCall, token: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        session["model_id"] = None
        session["model_name"] = ""
        session["model_picker_query"] = ""
        self._reset_search_state(session)
        await self._refresh_filters(call, token)

    async def _show_color_picker(self, call: InlineCall, token: str, page: int = 0):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        gift_id = session.get("gift_id")
        cached_colors = self._get_cached_colors(gift_id, allow_stale=True)
        if not cached_colors:
            await call.answer("получаю список цветов...")

        try:
            colors = cached_colors or await self._ensure_colors(gift_id)
        except Exception as e:
            await call.answer(
                f"не удалось получить список цветов: {utils.escape_html(str(e))[:120]}",
                show_alert=True,
            )
            return

        query = self._normalize_text(session.get("color_picker_query"))
        if query:
            filtered = [color for color in colors if query in self._normalize_text(color)]
        else:
            filtered = colors

        total_pages = max(1, (len(filtered) + COLOR_PAGE_SIZE - 1) // COLOR_PAGE_SIZE)
        page = min(max(int(page), 0), total_pages - 1)
        start = page * COLOR_PAGE_SIZE
        current = filtered[start : start + COLOR_PAGE_SIZE]

        rows = []
        for chunk_start in range(0, len(current), 2):
            row = []
            for color in current[chunk_start : chunk_start + 2]:
                label = color if len(color) <= 18 else color[:15] + "..."
                marker = (
                    "[x] "
                    if session.get("color_mode") == "exact"
                    and self._normalize_text(session.get("color_query"))
                    == self._normalize_text(color)
                    else ""
                )
                row.append(
                    _btn(
                        f"{marker}{label}",
                        ID_COLOR,
                        callback=self._set_color,
                        args=(token, color),
                    )
                )
            rows.append(row)

        nav = []
        if page > 0:
            nav.append(
                _btn(
                    "назад",
                    ID_BACK,
                    callback=self._show_color_picker,
                    args=(token, page - 1),
                    style="success",
                )
            )
        if page + 1 < total_pages:
            nav.append(
                _btn(
                    "вперёд",
                    ID_NEXT,
                    callback=self._show_color_picker,
                    args=(token, page + 1),
                    style="success",
                )
            )
        rows.append([_btn("любой цвет", ID_ANY, callback=self._clear_color, args=(token,))])
        if nav:
            rows.append(nav)
        rows.append(
            [
                _btn(
                    "к фильтрам",
                    ID_FOLDER,
                    callback=self._refresh_filters,
                    args=(token,),
                    style="primary",
                ),
                _btn("закрыть", ID_CLOSE, action="close", style="danger"),
            ]
        )

        gift_scope = utils.escape_html(session.get("gift_title") or "все NFT")
        selected = utils.escape_html(session.get("color_query") or "любой цвет")
        query_text = utils.escape_html(session.get("color_picker_query") or "без фильтра")
        count_text = f"{len(filtered)} шт." if filtered else "0"
        await call.edit(
            f"{E_COLOR} <b>Выбор цвета фона</b>\n\n"
            f"{E_NFT} <b>NFT:</b> <code>{gift_scope}</code>\n"
            f"{E_COLOR} <b>Сейчас:</b> <code>{selected}</code>\n"
            f"{E_SEARCH} <b>Фильтр списка:</b> <code>{query_text}</code>\n"
            f"{E_NOTE} <b>Найдено:</b> <code>{count_text}</code>\n"
            f"{E_FOLDER} <b>Страница:</b> <code>{page + 1}/{total_pages}</code>",
            reply_markup=rows,
        )

    async def _input_color_picker_query(self, call: InlineCall, query: str, token: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        value = " ".join((query or "").strip().split())
        if value.lower() in {"", "0", "any", "all", "любой", "все", "всё", "-"}:
            value = ""

        session["color_picker_query"] = value
        await self._show_color_picker(call, token, 0)

    async def _set_color(self, call: InlineCall, token: str, color: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        session["color_query"] = (color or "").strip()
        session["color_mode"] = "exact"
        self._reset_search_state(session)
        await self._refresh_filters(call, token)

    async def _clear_color(self, call: InlineCall, token: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        session["color_query"] = ""
        session["color_mode"] = "exact"
        session["color_picker_query"] = ""
        self._reset_search_state(session)
        await self._refresh_filters(call, token)

    def _reset_search_state(self, session: dict):
        session["results"] = []
        session["page"] = 0
        session["scan_states"] = []
        session["scan_complete"] = False
        session["seen_slugs"] = set()
        session["merge_lock"] = asyncio.Lock()
        session["search_lock"] = asyncio.Lock()

    async def _run_search(
        self, call: InlineCall, token: str, page: int = 0, reset: bool = False
    ):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        page = max(int(page), 0)
        if reset:
            self._reset_search_state(session)

        session["page"] = page
        await call.edit(
            f"{E_SEARCH} <b>Ищу подходящие подарки...</b>\n\n"
            f"{E_INFO} <i>Сканирую маркет Telegram. При широком фильтре это может занять время.</i>",
            reply_markup=[[_btn("закрыть", ID_CLOSE, action="close", style="danger")]],
        )

        needed = (page + 1) * RESULTS_PER_PAGE
        try:
            await self._ensure_results(session, needed)
        except Exception as e:
            await call.edit(
                f"{E_WARN} <b>Поиск не удался</b>\n\n"
                f"<code>{utils.escape_html(str(e))}</code>",
                reply_markup=[
                    [
                        _btn(
                            "к фильтрам",
                            ID_FOLDER,
                            callback=self._refresh_filters,
                            args=(token,),
                            style="primary",
                        ),
                        _btn("закрыть", ID_CLOSE, action="close", style="danger"),
                    ]
                ],
            )
            return

        await self._show_results(call, token)

    async def _show_results(self, call: InlineCall, token: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        results = session.get("results", [])
        page = int(session.get("page", 0) or 0)
        start = page * RESULTS_PER_PAGE
        chunk = results[start : start + RESULTS_PER_PAGE]

        if not chunk:
            await call.edit(
                self._render_no_results_text(session),
                reply_markup=[
                    [
                        _btn(
                            "к фильтрам",
                            ID_FOLDER,
                            callback=self._refresh_filters,
                            args=(token,),
                            style="primary",
                        ),
                        _btn(
                            "ещё раз",
                            ID_REFRESH,
                            callback=self._run_search,
                            args=(token, 0, True),
                            style="success",
                        ),
                    ],
                    [_btn("закрыть", ID_CLOSE, action="close", style="danger")],
                ],
            )
            return

        total = len(results)
        total_pages = max(1, (total + RESULTS_PER_PAGE - 1) // RESULTS_PER_PAGE)
        lines = [
            f"{E_OK} <b>Подходящие NFT-подарки</b>",
            "",
            f"{E_NOTE} <b>Фильтр:</b> {self._format_filters_line(session)}",
            f"{E_FOLDER} <b>Страница:</b> <code>{page + 1}</code>",
            "",
        ]

        for index, item in enumerate(chunk, start=start + 1):
            title = utils.escape_html(item["title"])
            kind = utils.escape_html(item["model_name"] or "не указан")
            backdrop = utils.escape_html(item["backdrop_name"] or "неизвестно")
            price = utils.escape_html(item["price_text"])
            price_unit = item.get("price_unit") or E_STAR
            url = utils.escape_html(item["url"])
            lines.extend(
                [
                    f"<b>{index}.</b> <b>{title}</b>",
                    f"{E_KIND} вид: <code>{kind}</code>",
                    f"{E_COLOR} фон: <code>{backdrop}</code>",
                    f"{E_MONEY} цена: <code>{price}</code> {price_unit}",
                    f"{E_LINK} {url}",
                    "",
                ]
            )

        if session.get("scan_complete"):
            lines.append(
                f"{E_INFO} <i>Найдено результатов: {total}. Показано страниц: {total_pages}.</i>"
            )
        else:
            lines.append(
                f"{E_INFO} <i>Сейчас собрано {total} результатов. Кнопка «другое» догрузит ещё.</i>"
            )

        markup = [
            [
                _btn(
                    "другое",
                    ID_NEXT,
                    callback=self._next_page,
                    args=(token,),
                )
            ],
            [
                _btn(
                    "к фильтрам",
                    ID_FOLDER,
                    callback=self._refresh_filters,
                    args=(token,),
                    style="primary",
                ),
                _btn(
                    "обновить",
                    ID_REFRESH,
                    callback=self._run_search,
                    args=(token, 0, True),
                    style="success",
                ),
            ],
            [_btn("закрыть", ID_CLOSE, action="close", style="danger")],
        ]

        if session.get("scan_complete") and start + RESULTS_PER_PAGE >= total:
            markup[0][0]["text"] = "сначала"
            markup[0][0]["emoji_id"] = str(ID_BACK)
            markup[0][0]["callback"] = self._run_search
            markup[0][0]["args"] = (token, 0, False)

        await call.edit("\n".join(lines).strip(), reply_markup=markup)

    async def _next_page(self, call: InlineCall, token: str):
        session = self._session_or_alert(token)
        if not session:
            await call.answer("сессия поиска уже протухла", show_alert=True)
            return

        session["page"] = int(session.get("page", 0) or 0) + 1
        needed = (session["page"] + 1) * RESULTS_PER_PAGE
        await self._ensure_results(session, needed)

        if session["page"] * RESULTS_PER_PAGE >= len(session.get("results", [])):
            session["page"] = 0

        await self._show_results(call, token)

    def _render_no_results_text(self, session: dict) -> str:
        return (
            f"{E_WARN} <b>Ничего не нашлось</b>\n\n"
            f"{E_NOTE} <b>Фильтр:</b> {self._format_filters_line(session)}\n\n"
            f"{E_INFO} <i>Попробуй поднять лимит цены, сменить цвет, выбрать другой вид или другое NFT.</i>"
        )

    def _format_filters_line(self, session: dict) -> str:
        parts = []
        gift_title = session.get("gift_title") or "любой NFT"
        parts.append(f"NFT <code>{utils.escape_html(gift_title)}</code>")

        model_name = session.get("model_name") or "любой вид"
        parts.append(f"вид <code>{utils.escape_html(model_name)}</code>")

        color = session.get("color_query") or "любой цвет"
        parts.append(f"цвет <code>{utils.escape_html(color)}</code>")

        currency = self._currency_text(session.get("currency"))
        parts.append(f"валюта <code>{utils.escape_html(currency)}</code>")

        max_price = session.get("max_price") or 0
        parts.append(
            f"до <code>{self._format_price(max_price)}</code>"
            if max_price
            else "<code>без лимита</code>"
        )
        return ", ".join(parts)

    async def _ensure_results(self, session: dict, needed_count: int):
        needed_count = max(RESULTS_PER_PAGE, min(int(needed_count), MAX_STORED_RESULTS))
        async with session["search_lock"]:
            if len(session["results"]) >= needed_count or session["scan_complete"]:
                return

            if not session["scan_states"]:
                await self._init_scan_states(session)

            await self._scan_until(session, needed_count)

    async def _init_scan_states(self, session: dict):
        catalog = await self._ensure_catalog()
        candidates = [
            item
            for item in catalog
            if item["resale_count"] > 0
            and (
                not session.get("gift_id")
                or int(item["id"]) == int(session["gift_id"])
            )
        ]

        if not candidates and session.get("gift_id"):
            candidates = [
                item for item in catalog if int(item["id"]) == int(session["gift_id"])
            ]

        exact_color = (
            session.get("color_query")
            if session.get("color_query") and session.get("color_mode") == "exact"
            else ""
        )
        if exact_color:
            candidates = await self._filter_candidates_by_exact_color(
                candidates, exact_color
            )

        candidates.sort(
            key=lambda item: (-int(item.get("resale_count", 0) or 0), item["title"].lower())
        )
        states = []
        for item in candidates:
            states.append(
                {
                    "gift_id": int(item["id"]),
                    "title": item["title"],
                    "next_offset": "",
                    "last_price": None,
                    "pages_done": 0,
                    "finished": False,
                    "started": False,
                    "resale_count": int(item.get("resale_count", 0) or 0),
                    "backdrop_id": item.get("exact_backdrop_id"),
                }
            )

        session["scan_states"] = states
        if not states:
            session["scan_complete"] = True

    async def _scan_until(self, session: dict, needed_count: int):
        while len(session["results"]) < needed_count and not session["scan_complete"]:
            threshold = self._current_threshold(session["results"], needed_count)
            active = []
            for state in session["scan_states"]:
                if state["finished"]:
                    continue
                if state["pages_done"] >= self._pages_per_gift_limit(session):
                    state["finished"] = True
                    continue

                if threshold is not None and state["last_price"] is not None:
                    if (
                        state["last_price"] > threshold
                        and len(session["results"]) >= needed_count
                    ):
                        state["finished"] = True
                        continue

                if state["started"] and not state["next_offset"]:
                    state["finished"] = True
                    continue

                active.append(state)

            if not active:
                session["scan_complete"] = True
                break

            active.sort(
                key=lambda state: (
                    state["pages_done"],
                    state["last_price"] if state["last_price"] is not None else -1,
                    -int(state.get("resale_count", 0) or 0),
                    state["title"].lower(),
                )
            )
            batch = active[:SCAN_BATCH_SIZE]
            await asyncio.gather(*(self._fetch_page(session, state) for state in batch))

        for state in session["scan_states"]:
            if (
                not state["finished"]
                and state["pages_done"] < self._pages_per_gift_limit(session)
                and (not state["started"] or state["next_offset"])
            ):
                return

        session["scan_complete"] = True

    def _pages_per_gift_limit(self, session: dict) -> int:
        limit = MAX_PAGES_PER_GIFT
        if session.get("color_query"):
            limit += 4
        if session.get("max_price"):
            limit += 2
        if session.get("gift_id"):
            limit += 2
        if session.get("model_id"):
            limit += 2
        if self._normalize_currency(session.get("currency")) == "ton":
            limit += 4
        return limit

    def _current_threshold(self, results: list, needed_count: int) -> Optional[float]:
        if len(results) < needed_count:
            return None
        currencies = {
            item.get("price_currency")
            for item in results[:needed_count]
            if item.get("price_currency")
        }
        if len(currencies) > 1:
            return None
        return results[needed_count - 1]["price_value"]

    async def _fetch_page(self, session: dict, state: dict):
        if state["finished"]:
            return

        request_kwargs = {
            "gift_id": state["gift_id"],
            "offset": state["next_offset"] or "",
            "limit": RESALE_FETCH_LIMIT,
            "sort_by_price": True,
        }
        if self._normalize_currency(session.get("currency")) == "stars":
            request_kwargs["stars_only"] = True

        attrs = []
        if state.get("backdrop_id") is not None:
            attrs.append(
                types.StarGiftAttributeIdBackdrop(backdrop_id=int(state["backdrop_id"]))
            )
        if session.get("model_id") is not None:
            attrs.append(
                types.StarGiftAttributeIdModel(document_id=int(session["model_id"]))
            )
        if attrs:
            request_kwargs["attributes"] = attrs

        response = await self._client(
            functions.payments.GetResaleStarGiftsRequest(**request_kwargs)
        )

        gifts = getattr(response, "gifts", []) or []
        parsed = []
        for gift in gifts:
            entry = self._parse_listing(session, state["title"], gift)
            if not entry:
                continue
            parsed.append(entry)

        async with session["merge_lock"]:
            fresh = []
            for entry in parsed:
                if entry["slug"] in session["seen_slugs"]:
                    continue
                session["seen_slugs"].add(entry["slug"])
                fresh.append(entry)

            if fresh:
                session["results"].extend(fresh)
                session["results"].sort(
                    key=lambda item: (
                        self._currency_rank(item.get("price_currency")),
                        item["price_value"],
                        item["title"].lower(),
                        item["slug"],
                    )
                )
                if len(session["results"]) > MAX_STORED_RESULTS:
                    session["results"] = session["results"][:MAX_STORED_RESULTS]

        state["started"] = True
        state["pages_done"] += 1
        state["next_offset"] = getattr(response, "next_offset", None) or ""
        state["last_price"] = self._page_last_price(gifts, session)
        if not state["next_offset"]:
            state["finished"] = True

    def _page_last_price(self, gifts: list, session: dict) -> Optional[float]:
        last_price = None
        for gift in gifts:
            price = self._extract_price(gift, session.get("currency"))
            if price is not None:
                last_price = price[0]
        return last_price

    def _parse_listing(self, session: dict, fallback_title: str, gift) -> Optional[dict]:
        slug = getattr(gift, "slug", None) or ""
        if not slug:
            return None

        title = (
            getattr(gift, "title", None) or fallback_title or ""
        ).strip() or fallback_title
        if session.get("gift_title") and title != session["gift_title"]:
            return None

        model_name, model_id = self._extract_model_info(gift)
        if session.get("model_id") and int(model_id or 0) != int(session["model_id"]):
            return None

        backdrop_name = self._extract_backdrop_name(gift)
        if not self._match_color(session, backdrop_name):
            return None

        price = self._extract_price(gift, session.get("currency"))
        if price is None:
            return None

        price_value, price_currency = price
        max_price = float(session.get("max_price") or 0)
        if max_price and price_value > max_price:
            return None

        return {
            "slug": slug,
            "title": title,
            "model_name": model_name,
            "model_id": model_id,
            "backdrop_name": backdrop_name,
            "price_value": price_value,
            "price_currency": price_currency,
            "price_text": self._format_price(price_value),
            "price_unit": self._price_unit(price_currency),
            "url": f"https://t.me/nft/{slug}",
        }

    def _extract_backdrop_name(self, gift) -> str:
        for attr in getattr(gift, "attributes", []) or []:
            if isinstance(attr, types.StarGiftAttributeBackdrop):
                return (getattr(attr, "name", None) or "").strip()
        return ""

    def _extract_model_info(self, gift) -> tuple:
        for attr in getattr(gift, "attributes", []) or []:
            if isinstance(attr, types.StarGiftAttributeModel):
                name = (getattr(attr, "name", None) or "").strip()
                document = getattr(attr, "document", None)
                model_id = int(getattr(document, "id", 0) or 0)
                return name, model_id
        return "", None

    def _match_color(self, session: dict, backdrop_name: str) -> bool:
        color_query = self._normalize_text(session.get("color_query"))
        if not color_query:
            return True

        backdrop = self._normalize_text(backdrop_name)
        if session.get("color_mode") == "exact":
            return backdrop == color_query
        return color_query in backdrop

    async def _filter_candidates_by_exact_color(
        self, candidates: List[dict], color_query: str
    ) -> List[dict]:
        if not candidates:
            return []

        target = self._normalize_text(color_query)
        semaphore = asyncio.Semaphore(DISCOVERY_BATCH_SIZE)

        async def _resolve(item: dict) -> Optional[dict]:
            async with semaphore:
                with contextlib.suppress(Exception):
                    for backdrop in await self._ensure_backdrops(int(item["id"])):
                        if self._normalize_text(backdrop["name"]) == target:
                            enriched = dict(item)
                            enriched["exact_backdrop_id"] = int(
                                backdrop["backdrop_id"]
                            )
                            return enriched
            return None

        resolved = await asyncio.gather(*(_resolve(item) for item in candidates))
        return [item for item in resolved if item]

    def _extract_price(self, gift, currency_mode: Optional[str]) -> Optional[tuple]:
        currency_mode = self._normalize_currency(currency_mode)
        resell_amount = getattr(gift, "resell_amount", None) or []
        amounts = [
            amount
            for amount in resell_amount
            if currency_mode == "any" or self._amount_currency(amount) == currency_mode
        ]
        first = amounts[0] if amounts else None
        if first is not None and hasattr(first, "amount"):
            currency = self._amount_currency(first)
            return self._amount_value(first, currency), currency

        if currency_mode in {"any", "stars"}:
            with contextlib.suppress(Exception):
                stars = getattr(gift, "stars", None)
                if stars is not None:
                    return float(stars), "stars"

            with contextlib.suppress(Exception):
                min_stars = getattr(gift, "resell_min_stars", None)
                if min_stars is not None:
                    return float(min_stars), "stars"

        return None

    def _amount_currency(self, amount) -> str:
        if isinstance(amount, types.StarsTonAmount):
            return "ton"
        return "stars"

    def _amount_value(self, amount, currency: str) -> float:
        raw = int(getattr(amount, "amount", 0) or 0)
        if currency == "ton":
            return raw / 1_000_000_000
        nanos = int(getattr(amount, "nanos", 0) or 0)
        return raw + nanos / 1_000_000_000

    def _price_unit(self, currency: str) -> str:
        return "TON" if currency == "ton" else E_STAR

    def _currency_rank(self, currency: Optional[str]) -> int:
        return {"stars": 0, "ton": 1}.get(currency or "", 2)

    def _format_price(self, value: float) -> str:
        if int(value) == value:
            return str(int(value))
        return f"{value:.3f}".rstrip("0").rstrip(".")

    def _normalize_text(self, value: Optional[str]) -> str:
        return " ".join((value or "").strip().lower().split())
