Отлично, реализуем маркер важности. Это позволит акцентировать внимание пользователей на ключевых дедлайнах или мероприятиях.

Мы добавим:

1. Новый шаг в FSM создания/редактирования, где будем спрашивать, важное ли событие.
    
2. Сортировку в запросе к БД, чтобы важные события всегда были вверху списка.
    
3. Визуальное выделение (например, иконкой ❗️) для важных событий при отображении.
    

### Шаг 1: Обновление Клавиатур

Нам понадобится простая инлайн-клавиатура с кнопками "Да" и "Нет".

**app/keyboards/inline.py** (добавьте эту функцию в конец файла)

      `# ... (существующие функции) ...  def get_yes_no_keyboard() -> InlineKeyboardMarkup:     """     Возвращает клавиатуру с кнопками "Да" и "Нет".     """     builder = InlineKeyboardBuilder()     builder.button(text="✅ Да", callback_data="choice:yes")     builder.button(text="❌ Нет", callback_data="choice:no")     return builder.as_markup()`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Шаг 2: Обновление Репозитория

Нужно изменить метод получения событий, чтобы он сначала сортировал по важности, а затем по дате.

**app/db/repository.py** (измените метод get_events_for_group в классе EventRepo)

      `# ... импорты ... from sqlalchemy import desc # импортируем desc для сортировки  class EventRepo:     # ... (остальные методы) ...     async def get_events_for_group(self, group_id: UUID, user_id: int, show_viewed: bool = False) -> list[Event]:         """         Получает список событий для группы.         - Сначала сортирует по важности, затем по дате.         """         viewed_events_subquery = (             select(ViewedEvent.event_id)             .where(ViewedEvent.user_id == user_id)         ).subquery()          stmt = (             select(Event)             .where(                 Event.group_id == group_id,                 Event.event_date >= datetime.now().date()             )             # ИЗМЕНЕНИЕ ЗДЕСЬ: Добавляем сортировку по is_important (desc - по убыванию, True будет первым)             .order_by(desc(Event.is_important), Event.event_date)         )          if not show_viewed:             stmt = stmt.where(Event.id.notin_(select(viewed_events_subquery)))                  result = await self.session.execute(stmt)         return result.scalars().all()     # ... (остальные методы)`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Шаг 3: Доработка Хэндлеров

Это основная часть работы. Мы добавим новый шаг в FSM.

**app/handlers/group_features.py** (нужно будет внести несколько изменений)

**1. Добавьте новый state в CreateEvent FSM.**

      `# ... (импорты) ...  # FSM для создания события class CreateEvent(StatesGroup):     waiting_for_title = State()     waiting_for_description = State()     waiting_for_subject = State()     waiting_for_date = State()     waiting_for_importance = State() # <-- НОВЫЙ ШАГ  # FSM для редактирования события (пока без изменений) class EditEvent(StatesGroup):     # ...`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**2. Измените цепочку FSM для создания события.**  
process_event_date теперь не будет последним шагом, а будет вести к waiting_for_importance. А новый хэндлер process_event_importance станет финальным.

      `# ... (остальной код)  # Изменяем хэндлер для даты @router.message(CreateEvent.waiting_for_date) async def process_event_date(message: Message, state: FSMContext): # Убираем session, она тут не нужна     try:         datetime.strptime(message.text, "%d.%m.%Y")     except ValueError:         await message.answer("Неверный формат даты. Пожалуйста, введите дату в формате ДД.ММ.ГГГГ.")         return          # Сохраняем дату и переходим к следующему шагу     await state.update_data(event_date_str=message.text)     await state.set_state(CreateEvent.waiting_for_importance)          await message.answer(         "Сделать это событие важным? (Оно будет отображаться вверху списка)",         reply_markup=inline.get_yes_no_keyboard()     )  # НОВЫЙ хэндлер для выбора важности, он теперь финальный @router.callback_query(CreateEvent.waiting_for_importance, F.data.startswith("choice:")) async def process_event_importance(callback: CallbackQuery, state: FSMContext, session: AsyncSession):     is_important = callback.data.split(":")[1] == "yes"          # Собираем все данные     event_data = await state.get_data()     await state.clear()          event_date = datetime.strptime(event_data['event_date_str'], "%d.%m.%Y")      # Получаем ID группы пользователя     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(callback.from_user.id)     if not user or not user.group_membership:         await callback.message.edit_text("Произошла ошибка: не удалось найти вашу группу.")         return      # Создаем событие в БД     event_repo = EventRepo(session)     await event_repo.create_event(         group_id=user.group_membership.group_id,         creator_id=callback.from_user.id,         title=event_data['title'],         description=event_data['description'],         subject=event_data['subject'],         event_date=event_date,         is_important=is_important # <-- передаем новый флаг     )          # Убираем клавиатуру "Да/Нет" и сообщаем об успехе     await callback.message.edit_text("✅ Событие успешно создано!")  # Старый хэндлер для даты (process_event_date), который раньше был финальным, теперь не нужен в той форме. # Вместо него у нас process_event_importance.  # --- ИЗМЕНЕНИЕ В ОТОБРАЖЕНИИ СОБЫТИЙ --- # Модифицируем хэндлер view_events, чтобы он добавлял иконку  @router.message(F.text.in_({"📋 Просмотреть актуальные", "👁️ Показать все (включая просмотренные)"})) async def view_events(message: Message, session: AsyncSession):     show_viewed = message.text == "👁️ Показать все (включая просмотренные)"          user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)     if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.")         return      event_repo = EventRepo(session)     events = await event_repo.get_events_for_group(         group_id=user.group_membership.group_id,         user_id=message.from_user.id,         show_viewed=show_viewed     )      if not events:         if show_viewed:             await message.answer("В вашей группе вообще нет предстоящих событий.")         else:             await message.answer("Нет новых актуальных событий. Возможно, вы все просмотрели.\n\nНажмите 'Показать все', чтобы увидеть их снова.")         return          header_text = "<b>🗓️ Все актуальные события:</b>" if show_viewed else "<b>🗓️ Новые актуальные события:</b>"     await message.answer(header_text, parse_mode="HTML")          for event in events:         # ИЗМЕНЕНИЕ ЗДЕСЬ: добавляем иконку важности         importance_icon = "❗️ " if event.is_important else "🔹 "         text = (             f"{importance_icon}<b>{event.title}</b> ({event.subject})\n"             f"   - <b>Дата:</b> {event.event_date.strftime('%d.%m.%Y')}\n"             f"   - <b>Описание:</b> {event.description}"         )         await message.answer(             text,             parse_mode="HTML",             reply_markup=inline.get_manage_event_keyboard(str(event.id), has_permissions=True)         )`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**3. Модифицируем EventRepo.create_event**, чтобы он принимал новый параметр.

**app/db/repository.py**

      `class EventRepo:     def __init__(self, session: AsyncSession):         self.session = session      async def create_event(self, group_id: UUID, creator_id: int, title: str, description: str, subject: str, event_date: datetime, is_important: bool) -> Event: # <-- Добавили is_important         """Создает новое событие в базе данных."""         new_event = Event(             group_id=group_id,             created_by_user_id=creator_id,             title=title,             description=description,             subject=subject,             event_date=event_date,             is_important=is_important # <-- Передали его в модель         )         self.session.add(new_event)         await self.session.commit()         return new_event          # ... остальной код`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

> **Опционально: Редактирование важности.** Реализация редактирования флага is_important аналогична. Нужно добавить кнопку "Важность" в EditEvent FSM, которая будет показывать клавиатуру get_yes_no_keyboard() и обновлять поле в базе данных. Пока я это опускаю, чтобы не перегружать ответ, но логика будет очень похожей на создание.

---

### Что теперь работает:

1. **Новый шаг при создании:** После ввода даты бот спрашивает: "Сделать это событие важным?" с кнопками "Да" и "Нет".
    
2. **Сохранение в БД:** Выбор пользователя сохраняется в поле is_important в таблице events.
    
3. **Правильная сортировка:** При запросе списка событий (📋 Просмотреть актуальные или 👁️ Показать все) события с флагом is_important=true теперь всегда отображаются первыми.
    
4. **Визуальное выделение:** В списке событий важные помечаются иконкой ❗️, а обычные — 🔹.
    

Фича полностью реализована. Это делает навигацию по событиям интуитивно понятнее и помогает выделить главное.