Отличный выбор! Это важный шаг, чтобы приложение стало полезным для всех членов группы, а не только для руководства.

Реализуем эту фичу. Мы добавим клавиатуру для обычного участника, создадим для него отдельный хэндлер и убедимся, что он видит события, но не может их редактировать.

### Шаг 1: Обновление Клавиатур

Нам нужна новая клавиатура для обычного участника и небольшое изменение в инлайн-клавиатурах.

**1. Файл app/keyboards/reply.py** (добавьте новую клавиатуру)

      `# ... (существующие клавиатуры main_menu_unregistered, main_menu_leader, events_menu) ...  # НОВАЯ клавиатура для обычного участника main_menu_member = ReplyKeyboardMarkup(     keyboard=[         [KeyboardButton(text="🗓️ Просмотреть события")],         # Здесь в будущем будут другие кнопки, например, "Мои темы", "Настройки"     ],     resize_keyboard=True )`
    

**2. Файл app/keyboards/inline.py** (добавим параметр в функцию)  
Мы изменим функцию get_manage_event_keyboard, чтобы она могла возвращать пустую клавиатуру, если у пользователя нет прав на управление.

      `# ... (существующие функции get_members_keyboard, get_manage_member_keyboard) ...  def get_manage_event_keyboard(event_id: str, has_permissions: bool) -> InlineKeyboardMarkup:     """     Клавиатура для управления конкретным событием.     Если has_permissions=False, вернет пустую клавиатуру.     """     builder = InlineKeyboardBuilder()          if has_permissions:         builder.button(             text="✏️ Редактировать",             callback_data=f"event:edit:{event_id}"         )         builder.button(             text="🗑️ Удалить",             callback_data=f"event:delete:{event_id}"         )          return builder.as_markup()`
    

IGNORE_WHEN_COPYING_START

content_copy download

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

### Шаг 2: Создание Хэндлера для Участников

Создадим новый файл для логики, доступной обычным участникам. Это поможет нам поддерживать порядок в проекте.

**1. Создайте новый файл app/handlers/group_member.py**

      `from aiogram import Router, F from aiogram.types import Message from sqlalchemy.ext.asyncio import AsyncSession  from app.db.repository import UserRepo, EventRepo from app.keyboards import inline  router = Router()  # Фильтр, который пропускает только обычных участников (не старост и не помощников) class MemberFilter(F):     async def __call__(self, message: Message, session: AsyncSession) -> bool:         user_repo = UserRepo(session)         user = await user_repo.get_user_with_group_info(message.from_user.id)         if user and user.group_membership:             # Пользователь является участником, если он в группе, но не староста и не помощник             return not user.group_membership.is_leader and not user.group_membership.is_assistant         return False  # Применяем фильтр ко всему роутеру router.message.filter(MemberFilter())  @router.message(F.text.in_({"🗓️ Просмотреть события", "📋 Просмотреть события"})) async def view_events_for_member(message: Message, session: AsyncSession):     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)      if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.")         return      event_repo = EventRepo(session)     events = await event_repo.get_events_for_group(user.group_membership.group_id)      if not events:         await message.answer("В вашей группе пока нет предстоящих событий.")         return          await message.answer("<b>🗓️ Актуальные события вашей группы:</b>", parse_mode="HTML")          # Отправляем каждое событие отдельным сообщением БЕЗ кнопок управления     for event in events:         text = (             f"🔹 <b>{event.title}</b> ({event.subject})\n"             f"   - <b>Дата:</b> {event.event_date.strftime('%d.%m.%Y')}\n"             f"   - <b>Описание:</b> {event.description}"         )         # Передаем has_permissions=False, чтобы клавиатура была пустой         await message.answer(             text,             parse_mode="HTML",             reply_markup=inline.get_manage_event_keyboard(str(event.id), has_permissions=False)         )`
    

IGNORE_WHEN_COPYING_START

content_copy download

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

> **Примечание:** Я добавил в F.text.in_ оба варианта текста кнопки ("🗓️ Просмотреть события" и "📋 Просмотреть события"), чтобы не дублировать код. В будущем лучше привести все кнопки к единому стилю.

### Шаг 3: Интеграция Новых Компонентов

Теперь нужно "подключить" все наши изменения к основному приложению.

**1. Файл app/handlers/common.py** (обновите cmd_start)  
Здесь мы добавим приветствие и клавиатуру для обычного участника.

      `# ... (импорты) from app.keyboards import reply # Убедитесь, что reply импортирован  @router.message(CommandStart()) async def cmd_start(message: Message, session: AsyncSession):     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)      if not user:         user = await user_repo.get_or_create_user(             telegram_id=message.from_user.id,             username=message.from_user.username,             first_name=message.from_user.first_name,             last_name=message.from_user.last_name         )      if user.group_membership:         if user.group_membership.is_leader:             await message.answer(                 f"Добро пожаловать, староста группы «{user.group_membership.group.name}»!",                 reply_markup=reply.main_menu_leader             )         elif user.group_membership.is_assistant:              await message.answer(                 f"Вы помощник старосты в группе «{user.group_membership.group.name}»",                 reply_markup=reply.main_menu_leader             )         else:              # ИЗМЕНЕНИЕ ЗДЕСЬ: показываем клавиатуру для участника              await message.answer(                  f"Вы участник группы «{user.group_membership.group.name}»",                  reply_markup=reply.main_menu_member              )     else:         await message.answer(             "👋 Добро пожаловать! Вы еще не состоите в группе.\n\nСоздайте свою или присоединитесь к существующей.",             reply_markup=reply.main_menu_unregistered         )  # ... (остальной код файла)`
    

IGNORE_WHEN_COPYING_START

content_copy download

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**2. Файл app/handlers/group_features.py** (обновите view_events)  
Теперь, когда у нас есть универсальная функция для клавиатуры, нужно передавать в нее права.

      `# ... (импорты)  # ... (код до view_events)  @router.message(F.text == "📋 Просмотреть события") async def view_events(message: Message, session: AsyncSession):     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)      if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.")         return      event_repo = EventRepo(session)     events = await event_repo.get_events_for_group(user.group_membership.group_id)      if not events:         await message.answer("В вашей группе пока нет предстоящих событий.")         return          await message.answer("<b>🗓️ Актуальные события вашей группы:</b>", parse_mode="HTML")          # ИЗМЕНЕНИЕ ЗДЕСЬ: передаем has_permissions=True     for event in events:         text = (             f"🔹 <b>{event.title}</b> ({event.subject})\n"             f"   - <b>Дата:</b> {event.event_date.strftime('%d.%m.%Y')}\n"             f"   - <b>Описание:</b> {event.description}"         )         await message.answer(             text,             parse_mode="HTML",             reply_markup=inline.get_manage_event_keyboard(str(event.id), has_permissions=True)         )  # ... (остальной код файла)`
    

IGNORE_WHEN_COPYING_START

content_copy download

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**3. Файл bot.py** (подключите новый роутер)

      `# ... (импорты) # Добавьте group_member в импорты from app.handlers import common, group_leader, group_features, group_member # ... (остальной код)  async def main():     # ...     # Регистрируем роутеры     dp.include_router(common.router)     dp.include_router(group_leader.router)     dp.include_router(group_features.router)     # Добавляем новый роутер для участников     dp.include_router(group_member.router)     # ...`
    

IGNORE_WHEN_COPYING_START

content_copy download

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

### Что теперь работает:

1. **Правильное приветствие:** Когда обычный участник (не староста и не помощник) вводит команду /start, он видит приветствие и свою собственную клавиатуру с кнопкой "🗓️ Просмотреть события".
    
2. **Просмотр событий:** При нажатии на эту кнопку срабатывает хэндлер из group_member.py.
    
3. **Отсутствие кнопок управления:** Хэндлер показывает участнику тот же список событий, что и старосте, но без инлайн-кнопок "Редактировать" и "Удалить".
    
4. **Сохранение прав для руководства:** Старосты и помощники по-прежнему видят события с кнопками управления, так как для них срабатывает хэндлер из group_features.py.
    
5. **Четкое разделение логики:** Код для обычных участников теперь находится в своем файле, что упрощает дальнейшую разработку.
    

Эта фича полностью реализована. Мы успешно разделили права на чтение и на управление контентом.