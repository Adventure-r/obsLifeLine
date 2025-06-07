Отлично, завершаем базовый цикл работы с событиями (CRUD: Create, Read, Update, Delete). Добавляем редактирование и удаление.

Это потребует использования инлайн-клавиатур, привязанных к конкретному сообщению с событием, и новой FSM для редактирования.

### Шаг 1: Обновление Клавиатур

Нам нужна инлайн-клавиатура для управления каждым отдельным событием.

**app/keyboards/inline.py** (добавьте эту функцию в конец файла)

      `# ... (существующие функции) ...  def get_manage_event_keyboard(event_id: str) -> InlineKeyboardMarkup:     """     Клавиатура для управления конкретным событием.     Принимает event_id в виде строки, т.к. UUID нужно будет передать в callback_data.     """     builder = InlineKeyboardBuilder()          builder.button(         text="✏️ Редактировать",         callback_data=f"event:edit:{event_id}"     )     builder.button(         text="🗑️ Удалить",         callback_data=f"event:delete:{event_id}"     )          return builder.as_markup()`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Шаг 2: Обновление Репозитория

Добавим методы для обновления и удаления событий.

**app/db/repository.py** (добавьте эти методы в класс EventRepo)

      `# ... импорты ...  class EventRepo:     # ... (существующие методы create_event, get_events_for_group) ...      async def get_event_by_id(self, event_id: UUID) -> Event | None:         """Находит событие по его ID."""         stmt = select(Event).where(Event.id == event_id)         result = await self.session.execute(stmt)         return result.scalar_one_or_none()              async def update_event(self, event_id: UUID, **kwargs) -> Event | None:         """         Обновляет данные события.         Принимает ID и поля для обновления (title, description, etc.).         """         event = await self.get_event_by_id(event_id)         if event:             for key, value in kwargs.items():                 if hasattr(event, key):                     setattr(event, key, value)             await self.session.commit()             return event         return None      async def delete_event(self, event_id: UUID) -> bool:         """Удаляет событие по его ID."""         event = await self.get_event_by_id(event_id)         if event:             await self.session.delete(event)             await self.session.commit()             return True         return False`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Шаг 3: Доработка Хэндлеров

Это основная часть. Мы изменим способ отображения событий и добавим логику для редактирования/удаления.

**app/handlers/group_features.py** (замените содержимое файла)

      `from aiogram import Router, F from aiogram.types import Message, CallbackQuery from aiogram.fsm.state import StatesGroup, State from aiogram.fsm.context import FSMContext from sqlalchemy.ext.asyncio import AsyncSession from datetime import datetime from uuid import UUID  from app.db.repository import UserRepo, EventRepo from app.keyboards import reply, inline  router = Router()  # FSM для создания события class CreateEvent(StatesGroup):     waiting_for_title = State()     waiting_for_description = State()     waiting_for_subject = State()     waiting_for_date = State()  # НОВЫЙ FSM для редактирования события class EditEvent(StatesGroup):     choosing_field = State()     writing_new_value = State()  # Фильтр, который пропускает только старост и помощников class LeaderOrAssistantFilter(F):     async def __call__(self, message: Message, session: AsyncSession) -> bool:         user_repo = UserRepo(session)         user = await user_repo.get_user_with_group_info(message.from_user.id)         if user and user.group_membership:             return user.group_membership.is_leader or user.group_membership.is_assistant         return False  # Применяем фильтр ко всему роутеру router.message.filter(LeaderOrAssistantFilter()) router.callback_query.filter(LeaderOrAssistantFilter())  # --- ОБЩИЕ КОМАНДЫ МЕНЮ --- @router.message(F.text == "📅 События") async def handle_events_menu(message: Message):     await message.answer("Вы в меню 'События'.", reply_markup=reply.events_menu)  @router.message(F.text == "⬅️ Назад в главное меню") async def handle_back_to_main_menu(message: Message, state: FSMContext, session: AsyncSession):     await state.clear() # На всякий случай сбрасываем состояние     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)     # Показываем правильную клавиатуру в зависимости от роли     if user and user.group_membership and user.group_membership.is_leader:         await message.answer("Вы вернулись в главное меню.", reply_markup=reply.main_menu_leader)     elif user and user.group_membership and user.group_membership.is_assistant:         await message.answer("Вы вернулись в главное меню.", reply_markup=reply.main_menu_leader) # У них такая же     else:         # Для обычных пользователей, если они как-то сюда попадут         await message.answer("Вы вернулись в главное меню.")  # --- ПРОСМОТР СОБЫТИЙ (ИЗМЕНЕНО) --- @router.message(F.text == "📋 Просмотреть события") async def view_events(message: Message, session: AsyncSession):     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)     if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.")         return      event_repo = EventRepo(session)     events = await event_repo.get_events_for_group(user.group_membership.group_id)      if not events:         await message.answer("В вашей группе пока нет предстоящих событий.")         return          await message.answer("<b>🗓️ Актуальные события вашей группы:</b>", parse_mode="HTML")     # Отправляем каждое событие отдельным сообщением с кнопками     for event in events:         text = (             f"🔹 <b>{event.title}</b> ({event.subject})\n"             f"   - <b>Дата:</b> {event.event_date.strftime('%d.%m.%Y')}\n"             f"   - <b>Описание:</b> {event.description}"         )         await message.answer(             text,             parse_mode="HTML",             reply_markup=inline.get_manage_event_keyboard(str(event.id))         )  # --- УДАЛЕНИЕ СОБЫТИЯ --- @router.callback_query(F.data.startswith("event:delete:")) async def delete_event_callback(callback: CallbackQuery, session: AsyncSession):     event_id = UUID(callback.data.split(":")[2])          event_repo = EventRepo(session)     success = await event_repo.delete_event(event_id)      if success:         # Просто удаляем сообщение с событием         await callback.message.delete()         await callback.answer("Событие удалено!", show_alert=True)     else:         await callback.answer("Не удалось удалить событие. Возможно, оно уже удалено.", show_alert=True)  # --- РЕДАКТИРОВАНИЕ СОБЫТИЯ (FSM) --- @router.callback_query(F.data.startswith("event:edit:")) async def edit_event_callback(callback: CallbackQuery, state: FSMContext):     event_id = callback.data.split(":")[2]     await state.update_data(event_id=event_id, message_id=callback.message.message_id)          # Создаем клавиатуру для выбора поля для редактирования     builder = inline.InlineKeyboardBuilder()     builder.button(text="Название", callback_data="edit_field:title")     builder.button(text="Описание", callback_data="edit_field:description")     builder.button(text="Предмет", callback_data="edit_field:subject")     builder.button(text="Дату", callback_data="edit_field:event_date")     builder.button(text="❌ Отмена", callback_data="edit_field:cancel")     builder.adjust(2)      await callback.message.edit_text(         "Какое поле вы хотите отредактировать?",         reply_markup=builder.as_markup()     )     await state.set_state(EditEvent.choosing_field)  @router.callback_query(EditEvent.choosing_field, F.data.startswith("edit_field:")) async def choose_field_to_edit(callback: CallbackQuery, state: FSMContext):     field = callback.data.split(":")[1]          if field == "cancel":         await state.clear()         # Нужно вернуть сообщение в исходное состояние, но это сложно.         # Проще просто сказать, что отменено, и удалить меню.         await callback.message.edit_text(callback.message.text) # Убираем кнопки         await callback.answer("Редактирование отменено.", show_alert=True)         return      await state.update_data(field_to_edit=field)     await state.set_state(EditEvent.writing_new_value)          prompt_text = {         "title": "Введите новое название:",         "description": "Введите новое описание:",         "subject": "Введите новый предмет:",         "event_date": "Введите новую дату (ДД.ММ.ГГГГ):"     }     await callback.message.edit_text(prompt_text.get(field, "Введите новое значение:"))  @router.message(EditEvent.writing_new_value) async def process_new_value(message: Message, state: FSMContext, session: AsyncSession):     data = await state.get_data()     event_id = UUID(data.get("event_id"))     field = data.get("field_to_edit")     new_value = message.text      # Валидация для даты     if field == "event_date":         try:             new_value = datetime.strptime(new_value, "%d.%m.%Y")         except ValueError:             await message.answer("Неверный формат даты. Пожалуйста, введите дату в формате ДД.ММ.ГГГГ.")             return      # Обновляем в БД     event_repo = EventRepo(session)     await event_repo.update_event(event_id, **{field: new_value})          await state.clear()     await message.answer("✅ Событие успешно обновлено!")          # После обновления нужно заново показать актуальные события     await view_events(message, session)  # --- СОЗДАНИЕ СОБЫТИЯ (FSM) - код остается без изменений, но я привожу его для полноты файла --- @router.message(F.text == "➕ Добавить событие") async def start_create_event(message: Message, state: FSMContext):     await state.set_state(CreateEvent.waiting_for_title)     await message.answer("Введите название события:")  @router.message(CreateEvent.waiting_for_title) async def process_event_title(message: Message, state: FSMContext):     await state.update_data(title=message.text)     await state.set_state(CreateEvent.waiting_for_description)     await message.answer("Отлично! Теперь введите описание события:")  @router.message(CreateEvent.waiting_for_description) async def process_event_description(message: Message, state: FSMContext):     await state.update_data(description=message.text)     await state.set_state(CreateEvent.waiting_for_subject)     await message.answer("Теперь введите название предмета/дисциплины:")  @router.message(CreateEvent.waiting_for_subject) async def process_event_subject(message: Message, state: FSMContext):     await state.update_data(subject=message.text)     await state.set_state(CreateEvent.waiting_for_date)     await message.answer("И последнее: введите дату события в формате ДД.ММ.ГГГГ (например, 31.12.2024):")  @router.message(CreateEvent.waiting_for_date) async def process_event_date(message: Message, state: FSMContext, session: AsyncSession):     try:         event_date = datetime.strptime(message.text, "%d.%m.%Y")     except ValueError:         await message.answer("Неверный формат даты. Пожалуйста, введите дату в формате ДД.ММ.ГГГГ.")         return      event_data = await state.get_data()     await state.clear()      user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)     if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.", reply_markup=reply.main_menu_leader)         return      event_repo = EventRepo(session)     await event_repo.create_event(         group_id=user.group_membership.group_id,         creator_id=message.from_user.id,         title=event_data['title'],         description=event_data['description'],         subject=event_data['subject'],         event_date=event_date     )     await message.answer("✅ Событие успешно создано!", reply_markup=reply.events_menu)`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

### Анализ реализованного и недостающего функционала по событиям

#### ✅ Что реализовано на данный момент:

1. **Создание событий (Create):** Старосты и помощники могут создавать события через пошаговый диалог.
    
2. **Чтение событий (Read):** Все участники группы (хотя пока мы реализовали это только в меню для старост/помощников) могут просматривать список актуальных событий. Каждое событие выводится отдельным сообщением.
    
3. **Редактирование событий (Update):** Старосты и помощники могут нажать кнопку "Редактировать" под любым событием, выбрать поле (название, описание, предмет, дата) и ввести новое значение.
    
4. **Удаление событий (Delete):** Старосты и помощники могут нажать кнопку "Удалить" под любым событием, и оно будет удалено из базы данных и из чата.
    
5. **Разграничение прав:** Все вышеперечисленные действия (кроме чтения) доступны только старостам и помощникам благодаря фильтру LeaderOrAssistantFilter.
    

#### ❌ Что еще НЕ реализовано согласно вашему ТЗ:

1. **Просмотр событий обычными участниками:** Сейчас кнопка "📋 Просмотреть события" находится в меню старосты/помощника. Нужно добавить аналогичную кнопку в меню обычного участника и реализовать для него хэндлер, который будет показывать события, но без кнопок управления.
    
2. **Прикрепление фото/видео:** Сейчас события только текстовые. Для добавления медиа нужно усложнить FSM создания/редактирования, запрашивая у пользователя отправку файла и сохраняя его file_id.
    
3. **Раздел "Просмотренные":** Нет логики для отметки событий как просмотренных. Для этого нужно:
    
    - Добавить инлайн-кнопку "✔️ Просмотрено" для обычных участников.
        
    - Реализовать хэндлер для этой кнопки, который будет создавать запись в таблице ViewedEvents.
        
    - Изменить логику отображения, чтобы не показывать уже просмотренные события в разделе "Актуальные" или помечать их как-то иначе.
        
4. **Маркер "Важное событие":** В модели есть поле is_important, но мы его никак не используем. Нужно добавить в FSM создания/редактирования вопрос "Сделать событие важным?" и при отображении выводить такие события первыми или выделять их.
    
5. **Связь с другими сущностями (Календарь/Бронь):** Самое главное. Сейчас "События" — это просто лента новостей. Весь функционал из вашего раздела "Теперь давай про бронь" ("Доклад по истории" "Защита Лабы у Зойкина") пока не реализован. Это включает в себя:
    
    - Визуальное отображение календаря.
        
    - Создание к событию привязанных списков тем для докладов.
        
    - Создание очередей на защиту.
        
    - Ограничение количества участников на тему/в очереди.
        
    - Возможность для участников выбирать тему или занимать место в очереди.
        

**Итог:** Мы построили надежный фундамент для работы с основной сущностью "Событие". Следующим логичным шагом будет реализация именно **связанных с событиями механик**, таких как **списки тем и очереди**, чтобы приблизиться к ключевой функциональности вашего приложения.