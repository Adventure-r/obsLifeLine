Отличный следующий шаг! Реализация событий — это сердце вашего приложения.

Я добавлю этот функционал, следуя нашей архитектуре. Мы сделаем так, чтобы староста (и в будущем помощник) мог создавать события через пошаговый диалог (используя FSM — машину состояний), а все участники группы могли их просматривать.

Вот код для каждого измененного или нового файла.

---

### Шаг 1: Обновление Моделей Базы Данных

Нам нужно добавить модель Event в наш файл с моделями.

**app/db/models.py** (добавьте этот код в конец файла)

      `# ... (существующие классы Base, User, Group, GroupMember) ...  class Event(Base):     __tablename__ = 'events'      id = Column(UUID(as_uuid=True), primary_key=True, server_default=func.uuid_generate_v4())     group_id = Column(UUID(as_uuid=True), ForeignKey('groups.id', ondelete='CASCADE'), nullable=False)     created_by_user_id = Column(BigInteger, ForeignKey('users.telegram_id', ondelete='SET NULL'))          title = Column(String(255), nullable=False)     description = Column(String, nullable=True)     subject = Column(String(255), nullable=True)     event_date = Column(DateTime(timezone=True), nullable=False)     is_important = Column(Boolean, default=False, nullable=False)          created_at = Column(DateTime(timezone=True), server_default=func.now())      # Связь для удобного доступа к группе из события     group = relationship("Group", back_populates="events")   # Также добавьте обратную связь в модель Group class Group(Base):     __tablename__ = 'groups'          id = Column(UUID(as_uuid=True), primary_key=True, server_default=func.uuid_generate_v4())     name = Column(String, nullable=False)     description = Column(String, nullable=True)     creator_id = Column(BigInteger, ForeignKey('users.telegram_id', ondelete='SET NULL'))     created_at = Column(DateTime(timezone=True), server_default=func.now())      # Связь с участниками     members = relationship("GroupMember", back_populates="group", cascade="all, delete-orphan")     # Новая связь с событиями     events = relationship("Event", back_populates="group", cascade="all, delete-orphan", order_by="Event.event_date")`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

> **Важно:** После изменения моделей вам нужно обновить структуру БД. Самый простой способ на данном этапе — удалить старые таблицы и заново выполнить SQL-скрипт из моего первого ответа (предварительно добавив в него создание таблицы events). В будущем для этого используется инструмент Alembic.

---

### Шаг 2: Расширение Репозитория

Добавим функции для создания и получения событий.

**app/db/repository.py** (добавьте новый класс EventRepo)

      `# ... (существующие импорты и классы UserRepo, GroupRepo) ... from app.db.models import Event from datetime import datetime  class EventRepo:     def __init__(self, session: AsyncSession):         self.session = session      async def create_event(self, group_id: UUID, creator_id: int, title: str, description: str, subject: str, event_date: datetime) -> Event:         """Создает новое событие в базе данных."""         new_event = Event(             group_id=group_id,             created_by_user_id=creator_id,             title=title,             description=description,             subject=subject,             event_date=event_date         )         self.session.add(new_event)         await self.session.commit()         return new_event      async def get_events_for_group(self, group_id: UUID) -> list[Event]:         """Получает список всех актуальных событий для группы, отсортированных по дате."""         stmt = (             select(Event)             .where(Event.group_id == group_id, Event.event_date >= datetime.now().date())             .order_by(Event.event_date)         )         result = await self.session.execute(stmt)         return result.scalars().all()`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

### Шаг 3: Обновление Клавиатур

Сделаем отдельное меню для событий, чтобы не загромождать главный экран.

**app/keyboards/reply.py** (замените содержимое файла)

      `from aiogram.types import ReplyKeyboardMarkup, KeyboardButton  # Клавиатура для пользователя, который не состоит в группе main_menu_unregistered = ReplyKeyboardMarkup(     keyboard=[         [KeyboardButton(text="🚀 Создать группу")],         [KeyboardButton(text="🔗 Присоединиться по ссылке")]     ],     resize_keyboard=True )  # Клавиатура для старосты main_menu_leader = ReplyKeyboardMarkup(     keyboard=[         [KeyboardButton(text="📅 События")],         [KeyboardButton(text="👥 Участники группы")],         [KeyboardButton(text="⚙️ Настройки группы")]     ],     resize_keyboard=True )  # Новая клавиатура для меню "События" events_menu = ReplyKeyboardMarkup(     keyboard=[         [KeyboardButton(text="📋 Просмотреть события"), KeyboardButton(text="➕ Добавить событие")],         [KeyboardButton(text="⬅️ Назад в главное меню")]     ],     resize_keyboard=True )`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

### Шаг 4: Реализация Логики в Хэндлерах

Это самая большая часть. Мы создадим FSM для пошагового создания события и обработчики для кнопок.

**app/handlers/group_leader.py** (замените содержимое файла)

      `from aiogram import Router, F from aiogram.types import Message from aiogram.fsm.state import StatesGroup, State from aiogram.fsm.context import FSMContext from sqlalchemy.ext.asyncio import AsyncSession from datetime import datetime  from app.db.repository import GroupRepo, UserRepo, EventRepo from app.keyboards import reply  router = Router()  # FSM для создания группы class CreateGroup(StatesGroup):     waiting_for_name = State()  # НОВЫЙ FSM для создания события class CreateEvent(StatesGroup):     waiting_for_title = State()     waiting_for_description = State()     waiting_for_subject = State()     waiting_for_date = State()  # --- ОБРАБОТЧИКИ ДЛЯ СОЗДАНИЯ ГРУППЫ (остаются без изменений) --- @router.message(CreateGroup.waiting_for_name) async def process_group_name(message: Message, state: FSMContext, session: AsyncSession):     group_name = message.text     if len(group_name) > 50:         await message.answer("Название слишком длинное. Пожалуйста, введите название до 50 символов.")         return      group_repo = GroupRepo(session)     await group_repo.create_group(name=group_name, creator_id=message.from_user.id)          await state.clear()     await message.answer(         f"🎉 Группа «{group_name}» успешно создана! Вы теперь её староста.",         reply_markup=reply.main_menu_leader     )  # --- НОВЫЕ ОБРАБОТЧИКИ ДЛЯ РАЗДЕЛА "СОБЫТИЯ" --- @router.message(F.text == "📅 События") async def handle_events_menu(message: Message):     await message.answer("Вы в меню 'События'.", reply_markup=reply.events_menu)  @router.message(F.text == "⬅️ Назад в главное меню") async def handle_back_to_main_menu(message: Message):     # В будущем здесь нужна будет проверка роли для показа правильной клавиатуры     await message.answer("Вы вернулись в главное меню.", reply_markup=reply.main_menu_leader)  @router.message(F.text == "➕ Добавить событие") async def start_create_event(message: Message, state: FSMContext):     await state.set_state(CreateEvent.waiting_for_title)     await message.answer("Введите название события:")  @router.message(CreateEvent.waiting_for_title) async def process_event_title(message: Message, state: FSMContext):     await state.update_data(title=message.text)     await state.set_state(CreateEvent.waiting_for_description)     await message.answer("Отлично! Теперь введите описание события:")  @router.message(CreateEvent.waiting_for_description) async def process_event_description(message: Message, state: FSMContext):     await state.update_data(description=message.text)     await state.set_state(CreateEvent.waiting_for_subject)     await message.answer("Теперь введите название предмета/дисциплины:")  @router.message(CreateEvent.waiting_for_subject) async def process_event_subject(message: Message, state: FSMContext):     await state.update_data(subject=message.text)     await state.set_state(CreateEvent.waiting_for_date)     await message.answer("И последнее: введите дату события в формате ДД.ММ.ГГГГ (например, 31.12.2024):")  @router.message(CreateEvent.waiting_for_date) async def process_event_date(message: Message, state: FSMContext, session: AsyncSession):     try:         event_date = datetime.strptime(message.text, "%d.%m.%Y")     except ValueError:         await message.answer("Неверный формат даты. Пожалуйста, введите дату в формате ДД.ММ.ГГГГ.")         return      event_data = await state.get_data()     await state.clear()      # Получаем ID группы пользователя     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)     if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.", reply_markup=reply.main_menu_leader)         return      # Создаем событие     event_repo = EventRepo(session)     await event_repo.create_event(         group_id=user.group_membership.group_id,         creator_id=message.from_user.id,         title=event_data['title'],         description=event_data['description'],         subject=event_data['subject'],         event_date=event_date     )      await message.answer("✅ Событие успешно создано!", reply_markup=reply.events_menu)  @router.message(F.text == "📋 Просмотреть события") async def view_events(message: Message, session: AsyncSession):     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)      if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.")         return      event_repo = EventRepo(session)     events = await event_repo.get_events_for_group(user.group_membership.group_id)      if not events:         await message.answer("В вашей группе пока нет предстоящих событий.")         return          response_text = "<b>🗓️ Актуальные события вашей группы:</b>\n\n"     for event in events:         response_text += (             f"🔹 <b>{event.title}</b> ({event.subject})\n"             f"   - <b>Дата:</b> {event.event_date.strftime('%d.%m.%Y')}\n"             f"   - <b>Описание:</b> {event.description}\n\n"         )          await message.answer(response_text, parse_mode="HTML")`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

### Что теперь работает:

1. Староста может нажать на кнопку "📅 События".
    
2. Ему откроется новое меню с кнопками "Просмотреть события" и "Добавить событие".
    
3. При нажатии на "Добавить событие" запустится пошаговый диалог, который запросит:
    
    - Название
        
    - Описание
        
    - Предмет
        
    - Дату
        
4. После ввода всех данных в базе данных будет создана новая запись в таблице events.
    
5. При нажатии на "Просмотреть события" бот покажет отформатированный список всех актуальных событий группы.
    
6. Кнопка "Назад" вернет пользователя в главное меню старосты.
    

### Что делать дальше:

- **Расширить права:** Сейчас создавать события может только староста. Нужно добавить проверку, чтобы это мог делать и помощник.
    
- **Редактирование/удаление:** Добавить к каждому событию в списке инлайн-кнопки "Редактировать" и "Удалить" (видимые только старосте/помощнику).
    
- **Календарь:** Реализовать визуальный календарь, о котором вы писали в ТЗ. Это сложная, но очень крутая фича.
    
- **Просмотренные события:** Добавить инлайн-кнопку "Просмотрено" для обычных участников и реализовать логику с таблицей ViewedEvents.