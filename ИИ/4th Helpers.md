Отлично! Реализуем систему ролей "Староста" -> "Помощник" -> "Участник". Это ключевой элемент управления группой.

Мы сделаем так, чтобы староста мог просмотреть список участников группы, выбрать любого и назначить/снять с него роль помощника. Помощники получат доступ к тем же функциям, что и старосты, за исключением управления составом группы.

### Шаг 1: Обновление Клавиатур

Нам понадобятся инлайн-клавиатуры для отображения списка участников и управления ими.

**app/keyboards/inline.py** (замените содержимое файла)

      `from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton from aiogram.utils.keyboard import InlineKeyboardBuilder from app.db.models import GroupMember from typing import List  # Callback-данные будем форматировать, чтобы передавать в них ID # Формат: "префикс:действие:id_сущности" # Например: "member:promote:12345678"  def get_members_keyboard(members: List[GroupMember]) -> InlineKeyboardMarkup:     builder = InlineKeyboardBuilder()     for member in members:         # Пропускаем старосту в списке на управление         if member.is_leader:             continue          # Определяем текст кнопки в зависимости от роли         role_icon = "🌟" if member.is_assistant else "👤"         user_name = member.user.first_name + (f" {member.user.last_name}" if member.user.last_name else "")                  builder.row(             InlineKeyboardButton(                 text=f"{role_icon} {user_name}",                 callback_data=f"member:manage:{member.user.telegram_id}"             )         )     return builder.as_markup()  def get_manage_member_keyboard(member_id: int, is_assistant: bool) -> InlineKeyboardMarkup:     builder = InlineKeyboardBuilder()          if is_assistant:         builder.button(             text="🚫 Снять с должности помощника",             callback_data=f"member:demote:{member_id}"         )     else:         builder.button(             text="🌟 Назначить помощником",             callback_data=f"member:promote:{member_id}"         )          builder.button(         text="🔥 Исключить из группы",         callback_data=f"member:kick:{member_id}"     )     builder.button(         text="⬅️ Назад к списку",         callback_data="member:back_to_list"     )          # Расставляем кнопки по рядам для красоты     builder.adjust(1)          return builder.as_markup()`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

### Шаг 2: Обновление Репозитория

Добавим функции для управления ролями и составом группы.

**app/db/repository.py** (добавьте эти методы в существующие классы)

      `# ... импорты ... from sqlalchemy.orm import selectinload  # В класс GroupRepo добавьте этот метод class GroupRepo:     # ... (существующий метод create_group) ...      async def get_group_with_members(self, group_id: UUID) -> Group | None:         """ Получает группу со всеми ее участниками и их профилями """         stmt = (             select(Group)             .options(                 selectinload(Group.members).selectinload(GroupMember.user)             )             .where(Group.id == group_id)         )         result = await self.session.execute(stmt)         return result.scalar_one_or_none()  # Создайте НОВЫЙ класс для управления участниками class MemberRepo:     def __init__(self, session: AsyncSession):         self.session = session          async def get_member_by_id(self, user_id: int) -> GroupMember | None:         """ Находит участника по его telegram_id """         stmt = (             select(GroupMember)             .options(selectinload(GroupMember.user)) # Загружаем и профиль пользователя             .where(GroupMember.user_id == user_id)         )         result = await self.session.execute(stmt)         return result.scalar_one_or_none()      async def set_assistant_role(self, user_id: int, is_assistant: bool) -> GroupMember | None:         """ Устанавливает или снимает роль помощника """         member = await self.get_member_by_id(user_id)         if member and not member.is_leader: # Нельзя изменить роль старосты             member.is_assistant = is_assistant             await self.session.commit()             return member         return None          async def remove_member(self, user_id: int) -> bool:         """ Удаляет участника из группы """         member = await self.get_member_by_id(user_id)         if member and not member.is_leader:             await self.session.delete(member)             await self.session.commit()             return True         return False`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

### Шаг 3: Разделение и Доработка Хэндлеров

Сейчас вся логика лежит в group_leader.py. Это неправильно. Помощник тоже должен иметь доступ к созданию событий. Давайте вынесем общий функционал и разграничим доступы.

**1. Создайте новый файл app/handlers/group_features.py**  
Сюда мы перенесем общий для старосты и помощника функционал (события).

**app/handlers/group_features.py**

      `from aiogram import Router, F from aiogram.types import Message, CallbackQuery from aiogram.fsm.state import StatesGroup, State from aiogram.fsm.context import FSMContext from sqlalchemy.ext.asyncio import AsyncSession from datetime import datetime  from app.db.repository import UserRepo, EventRepo from app.keyboards import reply  router = Router()  # FSM для создания события class CreateEvent(StatesGroup):     waiting_for_title = State()     waiting_for_description = State()     waiting_for_subject = State()     waiting_for_date = State()  # Фильтр, который пропускает только старост и помощников class LeaderOrAssistantFilter(F):     async def __call__(self, message: Message, session: AsyncSession) -> bool:         user_repo = UserRepo(session)         user = await user_repo.get_user_with_group_info(message.from_user.id)         if user and user.group_membership:             return user.group_membership.is_leader or user.group_membership.is_assistant         return False  # Применяем фильтр ко всему роутеру router.message.filter(LeaderOrAssistantFilter()) router.callback_query.filter(LeaderOrAssistantFilter())  @router.message(F.text == "📅 События") async def handle_events_menu(message: Message):     await message.answer("Вы в меню 'События'.", reply_markup=reply.events_menu)  @router.message(F.text == "➕ Добавить событие") async def start_create_event(message: Message, state: FSMContext):     await state.set_state(CreateEvent.waiting_for_title)     await message.answer("Введите название события:")  @router.message(CreateEvent.waiting_for_title) async def process_event_title(message: Message, state: FSMContext):     await state.update_data(title=message.text)     await state.set_state(CreateEvent.waiting_for_description)     await message.answer("Отлично! Теперь введите описание события:")  @router.message(CreateEvent.waiting_for_description) async def process_event_description(message: Message, state: FSMContext):     await state.update_data(description=message.text)     await state.set_state(CreateEvent.waiting_for_subject)     await message.answer("Теперь введите название предмета/дисциплины:")  @router.message(CreateEvent.waiting_for_subject) async def process_event_subject(message: Message, state: FSMContext):     await state.update_data(subject=message.text)     await state.set_state(CreateEvent.waiting_for_date)     await message.answer("И последнее: введите дату события в формате ДД.ММ.ГГГГ (например, 31.12.2024):")  @router.message(CreateEvent.waiting_for_date) async def process_event_date(message: Message, state: FSMContext, session: AsyncSession):     try:         event_date = datetime.strptime(message.text, "%d.%m.%Y")     except ValueError:         await message.answer("Неверный формат даты. Пожалуйста, введите дату в формате ДД.ММ.ГГГГ.")         return      event_data = await state.get_data()     await state.clear()      user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)     if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.")         return      event_repo = EventRepo(session)     await event_repo.create_event(         group_id=user.group_membership.group_id,         creator_id=message.from_user.id,         title=event_data['title'],         description=event_data['description'],         subject=event_data['subject'],         event_date=event_date     )      await message.answer("✅ Событие успешно создано!", reply_markup=reply.events_menu)  @router.message(F.text == "📋 Просмотреть события") async def view_events(message: Message, session: AsyncSession):     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)      if not user or not user.group_membership:         await message.answer("Произошла ошибка: не удалось найти вашу группу.")         return      event_repo = EventRepo(session)     events = await event_repo.get_events_for_group(user.group_membership.group_id)      if not events:         await message.answer("В вашей группе пока нет предстоящих событий.")         return          response_text = "<b>🗓️ Актуальные события вашей группы:</b>\n\n"     for event in events:         response_text += (             f"🔹 <b>{event.title}</b> ({event.subject})\n"             f"   - <b>Дата:</b> {event.event_date.strftime('%d.%m.%Y')}\n"             f"   - <b>Описание:</b> {event.description}\n\n"         )          await message.answer(response_text, parse_mode="HTML")`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**2. Измените app/handlers/group_leader.py**  
Теперь он будет отвечать только за то, что может делать староста: создание группы и управление участниками.

**app/handlers/group_leader.py** (замените содержимое файла)

      `from aiogram import Router, F from aiogram.types import Message, CallbackQuery from aiogram.fsm.state import StatesGroup, State from aiogram.fsm.context import FSMContext from sqlalchemy.ext.asyncio import AsyncSession  from app.db.repository import GroupRepo, UserRepo, MemberRepo from app.keyboards import reply, inline  router = Router()  # Фильтр, который пропускает ТОЛЬКО старост class LeaderFilter(F):     async def __call__(self, message: Message, session: AsyncSession) -> bool:         user_repo = UserRepo(session)         user = await user_repo.get_user_with_group_info(message.from_user.id)         return bool(user and user.group_membership and user.group_membership.is_leader)  # Применяем фильтр ко всем хэндлерам в этом файле router.message.filter(LeaderFilter()) router.callback_query.filter(LeaderFilter())  # FSM для создания группы class CreateGroup(StatesGroup):     waiting_for_name = State()  @router.message(CreateGroup.waiting_for_name) async def process_group_name(message: Message, state: FSMContext, session: AsyncSession):     # Этот хэндлер вызывается до того, как пользователь станет старостой,     # поэтому на него фильтр не действует, и это правильно.     group_name = message.text     group_repo = GroupRepo(session)     await group_repo.create_group(name=group_name, creator_id=message.from_user.id)          await state.clear()     await message.answer(         f"🎉 Группа «{group_name}» успешно создана! Вы теперь её староста.",         reply_markup=reply.main_menu_leader     )  # --- Управление участниками ---  @router.message(F.text == "👥 Участники группы") async def handle_group_members_list(message: Message, session: AsyncSession):     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)          group_repo = GroupRepo(session)     group = await group_repo.get_group_with_members(user.group_membership.group_id)      if not group or not group.members:         await message.answer("В вашей группе пока нет участников.")         return      await message.answer(         "Выберите участника для управления:",         reply_markup=inline.get_members_keyboard(group.members)     )  # Показать меню управления для конкретного участника @router.callback_query(F.data.startswith("member:manage:")) async def show_manage_member_menu(callback: CallbackQuery, session: AsyncSession):     member_id = int(callback.data.split(":")[2])     member_repo = MemberRepo(session)     member = await member_repo.get_member_by_id(member_id)      if not member:         await callback.answer("Участник не найден!", show_alert=True)         return          user_name = member.user.first_name + (f" {member.user.last_name}" if member.user.last_name else "")     text = f"Управление участником: <b>{user_name}</b>"          await callback.message.edit_text(         text,         reply_markup=inline.get_manage_member_keyboard(member_id, member.is_assistant),         parse_mode="HTML"     )  # Вернуться к списку участников @router.callback_query(F.data == "member:back_to_list") async def back_to_members_list(callback: CallbackQuery, session: AsyncSession):     # Копипастим логику из handle_group_members_list     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(callback.from_user.id)          group_repo = GroupRepo(session)     group = await group_repo.get_group_with_members(user.group_membership.group_id)      await callback.message.edit_text(         "Выберите участника для управления:",         reply_markup=inline.get_members_keyboard(group.members)     )  # Назначить/снять помощника @router.callback_query(F.data.startswith("member:promote:") | F.data.startswith("member:demote:")) async def toggle_assistant_role(callback: CallbackQuery, session: AsyncSession):     action, member_id_str = callback.data.split(":")[1:]     member_id = int(member_id_str)          member_repo = MemberRepo(session)     updated_member = await member_repo.set_assistant_role(member_id, is_assistant=(action == "promote"))          if updated_member:         await callback.answer(f"Роль успешно изменена!")         # Обновляем меню управления для этого участника         await show_manage_member_menu(callback, session)      else:         await callback.answer("Не удалось изменить роль.", show_alert=True)  # Исключить из группы @router.callback_query(F.data.startswith("member:kick:")) async def kick_member(callback: CallbackQuery, session: AsyncSession):     member_id = int(callback.data.split(":")[2])          member_repo = MemberRepo(session)     member = await member_repo.get_member_by_id(member_id)     user_name = member.user.first_name if member else "Неизвестный"          success = await member_repo.remove_member(member_id)          if success:         await callback.answer(f"Участник {user_name} исключен!", show_alert=True)         await back_to_members_list(callback, session) # Возвращаемся к обновленному списку     else:         await callback.answer("Не удалось исключить участника.", show_alert=True)`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**3. Обновите главный файл bot.py**, чтобы он знал о новом роутере.

**bot.py**

      `# ... импорты ... # Добавьте group_features в импорты from app.handlers import common, group_leader, group_features # ... остальной код ... async def main():     # ...     # Регистрируем роутеры     dp.include_router(common.router)     # Важно: роутер с более строгими правами (староста) должен идти ПЕРЕД роутером с общими правами     dp.include_router(group_leader.router)     dp.include_router(group_features.router) # Общий функционал для старосты и помощника     # ...`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**4. Обновите app/handlers/common.py** для правильного приветствия помощников.

      `# ... @router.message(CommandStart()) async def cmd_start(message: Message, session: AsyncSession):     # ... (код до if user.group_membership) ...     if user.group_membership:         if user.group_membership.is_leader:             await message.answer(                 f"Добро пожаловать, староста группы «{user.group_membership.group.name}»!",                 reply_markup=reply.main_menu_leader             )         # НОВАЯ ПРОВЕРКА         elif user.group_membership.is_assistant:              await message.answer(                 f"Вы помощник старосты в группе «{user.group_membership.group.name}»",                 reply_markup=reply.main_menu_leader # У них такая же клавиатура             )         else:              # Логика для обычного участника              await message.answer(f"Вы участник группы «{user.group_membership.group.name}»")     else:         # ...`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

### Что теперь работает:

1. **Разделение логики:** Функционал событий вынесен в отдельный файл group_features.py и доступен как старосте, так и помощнику.
    
2. **Управление участниками:** Староста может нажать кнопку "👥 Участники группы".
    
3. Ему откроется инлайн-список всех членов группы (кроме него самого). У помощников будет иконка 🌟.
    
4. При нажатии на участника откроется меню управления, где можно:
    
    - **Назначить помощником** (если он еще не помощник).
        
    - **Снять с должности** (если он уже помощник).
        
    - **Исключить из группы.**
        
5. После любого действия меню обновляется.
    
6. Пользователь, которого назначили помощником, при следующем нажатии /start или любой кнопки из главного меню будет обрабатываться роутером group_features.py и сможет, например, создавать события, но не сможет войти в раздел "Участники группы", т.к. этот хэндлер защищен фильтром LeaderFilter.
    

Это полноценная и расширяемая система ролей.