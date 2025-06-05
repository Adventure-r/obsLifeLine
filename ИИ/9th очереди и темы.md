Разумеется! Это ключевой шаг, который превращает ваше приложение из простой доски объявлений в мощный органайзер. Мы создадим "обвесы" для наших событий — списки тем и очереди.

Реализация будет идти по нашему проверенному пути: Модель -> Репозиторий -> Клавиатура -> Хэндлер.

### Шаг 1: Обновление Моделей и Репозитория

**1. Файл app/db/models.py** (добавьте новые классы и связи)  
Нам нужно описать в SQLAlchemy таблицы TopicLists, Topics и Queues.

      `# ... (существующие импорты и классы) ...  class Event(Base):     # ... (существующие поля) ...      # Связь для удобного доступа к группе из события     group = relationship("Group", back_populates="events")     # НОВЫЕ СВЯЗИ     topic_lists = relationship("TopicList", back_populates="event", cascade="all, delete-orphan")     queues = relationship("Queue", back_populates="event", cascade="all, delete-orphan")   class TopicList(Base):     __tablename__ = 'topiclists'      id = Column(UUID(as_uuid=True), primary_key=True, server_default=func.uuid_generate_v4())     event_id = Column(UUID(as_uuid=True), ForeignKey('events.id', ondelete="CASCADE"), nullable=False)     title = Column(String(255), nullable=False)     max_participants_per_topic = Column(INT, nullable=False)     created_at = Column(DateTime(timezone=True), server_default=func.now())          event = relationship("Event", back_populates="topic_lists")     topics = relationship("Topic", back_populates="topic_list", cascade="all, delete-orphan")  class Topic(Base):     __tablename__ = 'topics'      id = Column(UUID(as_uuid=True), primary_key=True, server_default=func.uuid_generate_v4())     topic_list_id = Column(UUID(as_uuid=True), ForeignKey('topiclists.id', ondelete="CASCADE"), nullable=False)     title = Column(String, nullable=False)      topic_list = relationship("TopicList", back_populates="topics")  class Queue(Base):     __tablename__ = 'queues'      id = Column(UUID(as_uuid=True), primary_key=True, server_default=func.uuid_generate_v4())     event_id = Column(UUID(as_uuid=True), ForeignKey('events.id', ondelete="CASCADE"), nullable=False)     title = Column(String, nullable=False)     description = Column(String, nullable=True)     max_participants = Column(INT, nullable=True) # Может быть NULL, если очередь бесконечная     created_at = Column(DateTime(timezone=True), server_default=func.now())          event = relationship("Event", back_populates="queues")  # Также нужно добавить обратную связь в модель Group class Group(Base):     # ...     events = relationship("Event", back_populates="group", cascade="all, delete-orphan", order_by="desc(Event.is_important), Event.event_date")`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

> **Важно:** Обновите вашу базу данных, добавив эти таблицы.

**2. Файл app/db/repository.py** (добавим методы для создания "фич")

      `# ... импорты ... from app.db.models import TopicList, Topic, Queue from typing import List from sqlalchemy.orm import selectinload  class EventRepo:     # ...     async def get_event_with_features(self, event_id: UUID) -> Event | None:         """Находит событие и сразу подгружает связанные с ним списки тем и очереди."""         stmt = (             select(Event)             .options(                 selectinload(Event.topic_lists).selectinload(TopicList.topics),                 selectinload(Event.queues)             )             .where(Event.id == event_id)         )         result = await self.session.execute(stmt)         return result.scalar_one_or_none()          # ...     async def create_topic_list(self, event_id: UUID, title: str, limit: int, topics: List[str]) -> TopicList:         """Создает список тем и сразу наполняет его темами."""         # Создаем контейнер         new_list = TopicList(             event_id=event_id,             title=title,             max_participants_per_topic=limit         )         self.session.add(new_list)         await self.session.flush() # Получаем ID до коммита          # Создаем сами темы         for topic_title in topics:             if topic_title.strip(): # Пропускаем пустые строки                 topic = Topic(topic_list_id=new_list.id, title=topic_title.strip())                 self.session.add(topic)                  await self.session.commit()         return new_list      async def create_queue(self, event_id: UUID, title: str, description: str, limit: int | None) -> Queue:         """Создает очередь."""         new_queue = Queue(             event_id=event_id,             title=title,             description=description,             max_participants=limit         )         self.session.add(new_queue)         await self.session.commit()         return new_queue`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Шаг 2: Обновление Клавиатур

Нам нужны кнопки, чтобы запускать процесс создания.

**app/keyboards/inline.py**

      `# ...  def get_manage_event_keyboard(event_id: str, has_permissions: bool) -> InlineKeyboardMarkup:     # ...     if has_permissions:         # ... (кнопки Edit, Delete)         # НОВАЯ КНОПКА         builder.button(             text="🔗 Привязать/Управлять",             callback_data=f"event:features:{event_id}"         )     # ...     if has_permissions:         builder.adjust(2, 1, 1) # Edit/Delete, Features, Viewed     # ...  def get_features_menu_keyboard(event_id: str) -> InlineKeyboardMarkup:     """Клавиатура для добавления 'фич' к событию."""     builder = InlineKeyboardBuilder()     builder.button(text="➕ Добавить список тем", callback_data=f"feature:add_topic_list:{event_id}")     builder.button(text="➕ Добавить очередь", callback_data=f"feature:add_queue:{event_id}")     builder.button(text="⬅️ Назад к событию", callback_data=f"feature:back_to_event:{event_id}")     builder.adjust(1)     return builder.as_markup()`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Шаг 3: Реализация Логики в Хэндлерах

Это самая объемная часть. Мы добавим новые FSM и хэндлеры.

**app/handlers/group_features.py**

      `# ... (импорты) ...  # --- FSM --- class CreateTopicList(StatesGroup):     waiting_for_title = State()     waiting_for_limit = State()     waiting_for_topics = State()  class CreateQueue(StatesGroup):     waiting_for_title = State()     waiting_for_description = State()     waiting_for_limit = State()  # ... (существующие хэндлеры до view_events) ...  # --- Вспомогательная функция для форматирования текста события --- async def format_event_message_text(event: Event) -> str:     """Формирует текст сообщения для события со всеми его 'фичами'."""     importance_icon = "❗️ " if event.is_important else "🔹 "     text = (         f"{importance_icon}<b>{event.title}</b> ({event.subject})\n"         f"   - <b>Дата:</b> {event.event_date.strftime('%d.%m.%Y')}\n"         f"   - <b>Описание:</b> {event.description}"     )      if event.topic_lists:         for topic_list in event.topic_lists:             text += f"\n\n📚 <b>Список тем: {topic_list.title}</b> (макс. {topic_list.max_participants_per_topic} чел./тема)"             if topic_list.topics:                 for i, topic in enumerate(topic_list.topics, 1):                     text += f"\n   {i}. {topic.title}"      if event.queues:         for queue in event.queues:             limit = f"(макс. {queue.max_participants} чел.)" if queue.max_participants else "(без лимита)"             text += f"\n\n👥 <b>Очередь: {queue.title}</b> {limit}"      return text  # --- Обновленный view_events --- @router.message(F.text.in_({"📋 Просмотреть актуальные", "👁️ Показать все (включая просмотренные)"})) async def view_events(message: Message, session: AsyncSession):     # ... (код для получения списка events остается без изменений) ...     for event in events:         event_with_features = await event_repo.get_event_with_features(event.id)         text = await format_event_message_text(event_with_features)                  await message.answer(             text,             parse_mode="HTML",             reply_markup=inline.get_manage_event_keyboard(str(event.id), has_permissions=True)         )  # --- Новые хэндлеры для управления "фичами" --- @router.callback_query(F.data.startswith("event:features:")) async def show_features_menu(callback: CallbackQuery):     event_id = callback.data.split(":")[2]     await callback.message.edit_text(         "Что вы хотите добавить или чем управлять?",         reply_markup=inline.get_features_menu_keyboard(event_id)     )  @router.callback_query(F.data.startswith("feature:back_to_event:")) async def back_to_event_view(callback: CallbackQuery, session: AsyncSession):     event_id = UUID(callback.data.split(":")[2])     event_repo = EventRepo(session)     event = await event_repo.get_event_with_features(event_id)     if not event:         await callback.message.edit_text("Событие не найдено.")         return              text = await format_event_message_text(event)     await callback.message.edit_text(         text,         parse_mode="HTML",         reply_markup=inline.get_manage_event_keyboard(str(event.id), has_permissions=True)     )  # --- Создание списка тем (FSM) --- @router.callback_query(F.data.startswith("feature:add_topic_list:")) async def start_add_topic_list(callback: CallbackQuery, state: FSMContext):     event_id = callback.data.split(":")[2]     await state.update_data(         event_id=event_id,          message_id=callback.message.message_id     )     await state.set_state(CreateTopicList.waiting_for_title)     await callback.message.edit_text("Введите название для списка тем (например, 'Темы докладов по Философии'):")  @router.message(CreateTopicList.waiting_for_title) async def process_topic_list_title(message: Message, state: FSMContext):     await state.update_data(title=message.text)     await state.set_state(CreateTopicList.waiting_for_limit)     await message.answer("Отлично! Теперь введите максимальное количество человек на одну тему (цифрой):")  @router.message(CreateTopicList.waiting_for_limit) async def process_topic_list_limit(message: Message, state: FSMContext):     if not message.text.isdigit() or int(message.text) < 1:         await message.answer("Пожалуйста, введите целое положительное число.")         return     await state.update_data(limit=int(message.text))     await state.set_state(CreateTopicList.waiting_for_topics)     await message.answer("Теперь отправьте список тем. Каждая тема - с новой строки.")  @router.message(CreateTopicList.waiting_for_topics) async def process_topic_list_topics(message: Message, state: FSMContext, session: AsyncSession):     data = await state.get_data()     state.clear()          event_id = UUID(data['event_id'])     topics = message.text.split('\n')          event_repo = EventRepo(session)     await event_repo.create_topic_list(         event_id=event_id,         title=data['title'],         limit=data['limit'],         topics=topics     )          # Обновляем оригинальное сообщение о событии     updated_event = await event_repo.get_event_with_features(event_id)     text = await format_event_message_text(updated_event)      # Редактируем сообщение, на котором были кнопки управления     await message.bot.edit_message_text(         text=text,         chat_id=message.chat.id,         message_id=data['message_id'],         parse_mode="HTML",         reply_markup=inline.get_manage_event_keyboard(str(event_id), has_permissions=True)     )     await message.answer("✅ Список тем успешно добавлен!")  # ... (Остальной код файла: FSM для создания и редактирования событий) # FSM для создания ОЧЕРЕДИ реализуется абсолютно аналогично, я оставлю это как упражнение. # Нужно будет создать хэндлеры для CreateQueue FSM, которые по очереди запрашивают # title, description и limit, а затем вызывают event_repo.create_queue.`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Что теперь работает:

1. **Новая кнопка:** Под каждым событием у старосты/помощника появилась кнопка "🔗 Привязать/Управлять".
    
2. **Меню "фич":** При нажатии на нее открывается новое меню, где можно выбрать "➕ Добавить список тем".
    
3. **Создание списка тем:** Запускается пошаговый диалог:
    
    - Бот спрашивает название списка (например, "Доклады по Экономике").
        
    - Бот спрашивает лимит участников на тему.
        
    - Бот просит прислать сами темы, каждая с новой строки.
        
4. **Сохранение в БД:** После получения всех данных в базе создается запись в topiclists и связанные с ней записи в topics.
    
5. **Обновление отображения:** Самое главное — после создания списка тем оригинальное сообщение с событием **обновляется**, и в его тексте появляется отформатированный список добавленных тем.
    

### Что дальше?

- **Реализовать FSM для создания Очереди.** Логика будет очень похожей на FSM для списка тем.
    
- **Реализовать для участников возможность выбора.** Добавить к каждой теме и в очередь инлайн-кнопку "Выбрать" / "Записаться" и реализовать хэндлеры для нее. Это будет следующий большой шаг.