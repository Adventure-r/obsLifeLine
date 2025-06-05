Отличная идея! Эта фича сделает ленту событий гораздо удобнее для пользователей, позволяя им скрывать то, с чем они уже ознакомились.

Мы реализуем это так:

1. Добавим кнопку "✔️ Просмотрено" к событиям для всех участников.
    
2. При нажатии на нее событие будет "исчезать" из ленты (мы просто будем его скрывать при следующем запросе).
    
3. Добавим в меню событий переключатель "Показать просмотренные", чтобы пользователь мог вернуть скрытые события.
    

### Шаг 1: Обновление Моделей и Репозитория

Сначала убедимся, что у нас есть все необходимое в базе данных.

**1. Файл app/db/models.py** (добавьте этот класс)

      `# ... (существующие классы) ...  class ViewedEvent(Base):     __tablename__ = 'viewedevents'      # Композитный первичный ключ, чтобы не было дубликатов     user_id = Column(BigInteger, ForeignKey('users.telegram_id', ondelete='CASCADE'), primary_key=True)     event_id = Column(UUID(as_uuid=True), ForeignKey('events.id', ondelete='CASCADE'), primary_key=True)          viewed_at = Column(DateTime(timezone=True), server_default=func.now())`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

> **Важно:** Не забудьте добавить эту таблицу в вашу базу данных. Проще всего — добавить соответствующий CREATE TABLE в ваш SQL-скрипт и пересоздать таблицы.

**2. Файл app/db/repository.py** (обновите EventRepo)  
Нам нужно научить репозиторий работать с просмотренными событиями.

      `# ... импорты ... from app.db.models import Event, ViewedEvent from sqlalchemy import and_  class EventRepo:     # ... (существующие методы) ...      async def get_events_for_group(self, group_id: UUID, user_id: int, show_viewed: bool = False) -> list[Event]:         """         Получает список событий для группы.         - user_id: нужен, чтобы отфильтровать просмотренные.         - show_viewed: если True, покажет все события, иначе - только непросмотренные.         """         # Находим ID событий, которые пользователь уже просмотрел         viewed_events_subquery = (             select(ViewedEvent.event_id)             .where(ViewedEvent.user_id == user_id)         ).subquery()          # Основной запрос         stmt = (             select(Event)             .where(                 Event.group_id == group_id,                 Event.event_date >= datetime.now().date()             )             .order_by(Event.event_date)         )          if not show_viewed:             # Если не нужно показывать просмотренные, исключаем их             stmt = stmt.where(Event.id.notin_(select(viewed_events_subquery)))                  result = await self.session.execute(stmt)         return result.scalars().all()      async def mark_event_as_viewed(self, user_id: int, event_id: UUID):         """Отмечает событие как просмотренное для конкретного пользователя."""         # Проверяем, нет ли уже такой записи         existing = await self.session.get(ViewedEvent, (user_id, event_id))         if not existing:             view = ViewedEvent(user_id=user_id, event_id=event_id)             self.session.add(view)             await self.session.commit()`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Шаг 2: Обновление Клавиатур

Добавим кнопку "Просмотрено" и переключатель для показа скрытых событий.

**1. Файл app/keyboards/inline.py** (измените get_manage_event_keyboard)

      `# ... (импорты) ...  def get_manage_event_keyboard(event_id: str, has_permissions: bool) -> InlineKeyboardMarkup:     """     Клавиатура для управления конкретным событием.     Добавляет кнопку "Просмотрено" для всех.     """     builder = InlineKeyboardBuilder()          # Кнопки управления для старост/помощников     if has_permissions:         builder.button(             text="✏️ Редактировать",             callback_data=f"event:edit:{event_id}"         )         builder.button(             text="🗑️ Удалить",             callback_data=f"event:delete:{event_id}"         )          # Кнопка "Просмотрено" для всех     builder.button(         text="✔️ Просмотрено",         callback_data=f"event:view:{event_id}"     )          # Расставляем кнопки красиво     if has_permissions:         builder.adjust(2, 1) # 2 кнопки управления в ряд, 1 кнопка просмотра в ряд          return builder.as_markup()`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**2. Файл app/keyboards/reply.py** (измените events_menu)

      `# ... (существующие клавиатуры) ...  # Клавиатура для меню "События" events_menu = ReplyKeyboardMarkup(     keyboard=[         [KeyboardButton(text="📋 Просмотреть актуальные")],         [KeyboardButton(text="➕ Добавить событие")], # Эта кнопка будет видна всем, но сработает только у тех, у кого есть права         [KeyboardButton(text="👁️ Показать все (включая просмотренные)")],         [KeyboardButton(text="⬅️ Назад в главное меню")]     ],     resize_keyboard=True )`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Шаг 3: Обновление Хэндлеров

Теперь самое интересное — заставим все это работать вместе.

**1. Файл app/handlers/group_features.py** (для старост и помощников)

      `# ... (импорты) ... # Добавляем CallbackQuery в импорты aiogram from aiogram.types import Message, CallbackQuery  # ... (FSM и фильтры) ...  # --- ПРОСМОТР СОБЫТИЙ (ИЗМЕНЕНО) --- # Этот хэндлер теперь будет обрабатывать оба варианта просмотра @router.message(F.text.in_({"📋 Просмотреть актуальные", "👁️ Показать все (включая просмотренные)"})) async def view_events(message: Message, session: AsyncSession):     show_viewed = message.text == "👁️ Показать все (включая просмотренные)"          user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)     if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.")         return      event_repo = EventRepo(session)     events = await event_repo.get_events_for_group(         group_id=user.group_membership.group_id,         user_id=message.from_user.id,         show_viewed=show_viewed     )      if not events:         if show_viewed:             await message.answer("В вашей группе вообще нет предстоящих событий.")         else:             await message.answer("Нет новых актуальных событий. Возможно, вы все просмотрели.\n\nНажмите 'Показать все', чтобы увидеть их снова.")         return          header_text = "<b>🗓️ Все актуальные события:</b>" if show_viewed else "<b>🗓️ Новые актуальные события:</b>"     await message.answer(header_text, parse_mode="HTML")          for event in events:         text = (             f"🔹 <b>{event.title}</b> ({event.subject})\n"             f"   - <b>Дата:</b> {event.event_date.strftime('%d.%m.%Y')}\n"             f"   - <b>Описание:</b> {event.description}"         )         await message.answer(             text,             parse_mode="HTML",             reply_markup=inline.get_manage_event_keyboard(str(event.id), has_permissions=True)         )  # НОВЫЙ обработчик для кнопки "Просмотрено" # Он будет общим для всех, поэтому разместим его здесь @router.callback_query(F.data.startswith("event:view:")) async def view_event_callback(callback: CallbackQuery, session: AsyncSession):     event_id = UUID(callback.data.split(":")[2])          event_repo = EventRepo(session)     await event_repo.mark_event_as_viewed(         user_id=callback.from_user.id,         event_id=event_id     )          # Просто удаляем сообщение с событием из чата, чтобы не мешало     await callback.message.delete()     await callback.answer("Отмечено как просмотренное!")  # ... (остальной код файла без изменений) ...`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**2. Файл app/handlers/group_member.py** (для обычных участников)  
Здесь мы просто удалим старый хэндлер и позволим участникам использовать общий.

      `from aiogram import Router, F from aiogram.types import Message, CallbackQuery # Добавляем CallbackQuery from sqlalchemy.ext.asyncio import AsyncSession  from app.db.repository import UserRepo, EventRepo from app.keyboards import inline, reply # Добавляем reply  router = Router()  # Фильтр, который пропускает только обычных участников class MemberFilter(F):     async def __call__(self, event: Message | CallbackQuery, session: AsyncSession) -> bool:         user_repo = UserRepo(session)         user = await user_repo.get_user_with_group_info(event.from_user.id)         if user and user.group_membership:             return not user.group_membership.is_leader and not user.group_membership.is_assistant         return False  # Применяем фильтр ко всему роутеру router.message.filter(MemberFilter()) router.callback_query.filter(MemberFilter()) # И к колбэкам тоже  # Этот хэндлер теперь не нужен, так как мы сделали общий в group_features.py # Но нам нужно меню событий для обычных участников @router.message(F.text == "🗓️ Просмотреть события") async def handle_events_menu_for_member(message: Message):     # Убираем кнопку "Добавить событие" из меню для обычных участников     member_events_menu = reply.ReplyKeyboardMarkup(         keyboard=[             [reply.KeyboardButton(text="📋 Просмотреть актуальные")],             [reply.KeyboardButton(text="👁️ Показать все (включая просмотренные)")],             [reply.KeyboardButton(text="⬅️ Назад в главное меню")]         ],         resize_keyboard=True     )     await message.answer("Вы в меню 'События'.", reply_markup=member_events_menu)`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**3. Файл bot.py** (порядок роутеров)  
Порядок регистрации роутеров становится критически важным. Общий функционал должен идти после специфичного.

      `# ... (импорты) ... from app.handlers import common, group_leader, group_features, group_member  # ... (код) ... async def main():     # ...     # Регистрируем роутеры     dp.include_router(common.router)     dp.include_router(group_leader.router) # Специфичные права старосты     dp.include_router(group_member.router) # Специфичные права участника     dp.include_router(group_features.router) # Общий функционал для всех, кто в группе     # ...`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

> **Почему такой порядок?** aiogram проверяет роутеры по порядку. Если бы group_features (который ловит всех, у кого есть права) стоял раньше group_member, то хэндлеры из group_member никогда бы не сработали, так как помощники и старосты тоже проходят фильтр MemberFilter (он проверяет is_leader=False и is_assistant=False). Теперь же сначала отработают специфичные роутеры, а если там совпадений не нашлось, проверка пойдет в group_features.

---

### Что теперь работает:

1. **Новое меню событий:** У всех пользователей в меню событий теперь есть кнопки "Просмотреть актуальные" и "Показать все".
    
2. **Кнопка "Просмотрено":** Под каждым событием есть кнопка "✔️ Просмотрено".
    
3. **Скрытие событий:** При нажатии на "✔️ Просмотрено" в базу добавляется запись, а сообщение с событием удаляется из чата.
    
4. **Фильтрация:** При нажатии "Просмотреть актуальные" пользователь видит только те события, которые он еще не отмечал как просмотренные.
    
5. **Показ всех:** При нажатии "Показать все" пользователь видит полный список актуальных событий, включая те, что он уже просмотрел.
    
6. **Сохранение прав:** Старосты и помощники по-прежнему видят кнопки "Редактировать" и "Удалить", а обычные участники — нет.
    

Эта фича полностью реализована и делает взаимодействие с лентой событий гораздо более интерактивным и удобным.