Это не будет полностью готовый продукт со всем функционалом, так как это работа на недели, но это будет **прочный фундамент**, на котором вы сможете строить дальше. Все файлы будут на своих местах, основная логика (регистрация, создание группы) будет работать, а для остального будут созданы "заглушки", чтобы вы четко видели, где и что дописывать.

Просто скопируйте эти файлы в соответствующие папки, настройте .env и запускайте.

---

### Шаг 1: Структура проекта (напоминание)

      `/study_group_bot ├── .env ├── requirements.txt ├── bot.py │ └── app/     ├── __init__.py     ├── config.py     ├── handlers/     │   ├── __init__.py     │   ├── admin.py     │   ├── common.py     │   ├── group_assistant.py     │   ├── group_leader.py     │   └── group_member.py     ├── keyboards/     │   ├── __init__.py     │   ├── inline.py     │   └── reply.py     ├── middlewares/     │   ├── __init__.py     │   └── db.py     ├── db/     │   ├── __init__.py     │   ├── models.py     │   └── repository.py     └── utils/         ├── __init__.py         └── tools.py`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487).

IGNORE_WHEN_COPYING_END

---

### Шаг 2: Код для каждого файла

#### 1. Корневая директория

**requirements.txt**

      `aiogram==3.3.0 sqlalchemy[asyncio]==2.0.25 asyncpg==0.29.0 alembic==1.13.1 python-dotenv==1.0.0`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Text

IGNORE_WHEN_COPYING_END

**.env** (создайте этот файл и вставьте свои данные)

      `# Вставьте сюда токен вашего бота, полученный от @BotFather BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11  # Строка для подключения к вашей базе данных PostgreSQL # Формат: postgresql+asyncpg://<user>:<password>@<host>:<port>/<dbname> DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/stud_bot_db`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Env

IGNORE_WHEN_COPYING_END

**bot.py** (Главный файл для запуска)

      `import asyncio import logging from os import getenv  from aiogram import Bot, Dispatcher from aiogram.fsm.storage.memory import MemoryStorage from dotenv import load_dotenv from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker  # Импортируем роутеры из handlers from app.handlers import common, group_leader from app.middlewares.db import DbSessionMiddleware  # Загружаем переменные окружения из .env load_dotenv()  # Настройка логирования logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(name)s - %(message)s') logger = logging.getLogger(__name__)  async def main():     logger.info("Starting bot...")      # Создаем асинхронный движок для SQLAlchemy     engine = create_async_engine(getenv("DATABASE_URL"), echo=False) # echo=True для отладки SQL-запросов          # Создаем фабрику сессий     session_maker = async_sessionmaker(engine, expire_on_commit=False)      # Создаем объекты Бота и Диспетчера     bot = Bot(token=getenv("BOT_TOKEN"))     dp = Dispatcher(storage=MemoryStorage())      # Регистрируем middleware для передачи сессии в хэндлеры     # Это должно быть сделано перед регистрацией роутеров     dp.update.middleware(DbSessionMiddleware(session_pool=session_maker))      # Регистрируем роутеры     dp.include_router(common.router)     dp.include_router(group_leader.router)     # ... здесь будут регистрироваться другие роутеры (admin, group_assistant, etc.) ...      # Удаляем вебхук и запускаем polling     await bot.delete_webhook(drop_pending_updates=True)     try:         await dp.start_polling(bot)     finally:         await bot.session.close()         await engine.dispose()  if __name__ == "__main__":     try:         asyncio.run(main())     except (KeyboardInterrupt, SystemExit):         logger.info("Bot stopped!")`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

#### 2. Директория app/

**app/config.py**

      `from os import getenv from dotenv import load_dotenv  load_dotenv()  class Settings:     BOT_TOKEN = getenv("BOT_TOKEN")     DATABASE_URL = getenv("DATABASE_URL")  settings = Settings()`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

#### 3. Директория app/db/

**app/db/models.py** (ORM-модели, описывающие таблицы)

      `from sqlalchemy import (     BigInteger, String, Column, DateTime, func, JSONB, Boolean,     UUID, ForeignKey, INT ) from sqlalchemy.orm import DeclarativeBase, relationship  class Base(DeclarativeBase):     pass  class User(Base):     __tablename__ = 'users'      telegram_id = Column(BigInteger, primary_key=True)     telegram_username = Column(String, nullable=True)     first_name = Column(String, nullable=False)     last_name = Column(String, nullable=True)     middle_name = Column(String, nullable=True)     created_at = Column(DateTime(timezone=True), server_default=func.now())     last_active_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())     notification_settings = Column(JSONB, nullable=True)      # Связь с участником группы     group_membership = relationship("GroupMember", back_populates="user", uselist=False, cascade="all, delete-orphan")      def __repr__(self):         return f"<User(id={self.telegram_id}, name='{self.first_name}')>"  class Group(Base):     __tablename__ = 'groups'          id = Column(UUID(as_uuid=True), primary_key=True, server_default=func.uuid_generate_v4())     name = Column(String, nullable=False)     description = Column(String, nullable=True)     creator_id = Column(BigInteger, ForeignKey('users.telegram_id', ondelete='SET NULL'))     created_at = Column(DateTime(timezone=True), server_default=func.now())      # Связь с участниками     members = relationship("GroupMember", back_populates="group", cascade="all, delete-orphan")  class GroupMember(Base):     __tablename__ = 'groupmembers'      id = Column(UUID(as_uuid=True), primary_key=True, server_default=func.uuid_generate_v4())     user_id = Column(BigInteger, ForeignKey('users.telegram_id', ondelete='CASCADE'), unique=True, nullable=False)     group_id = Column(UUID(as_uuid=True), ForeignKey('groups.id', ondelete='CASCADE'), nullable=False)     is_leader = Column(Boolean, default=False, nullable=False)     is_assistant = Column(Boolean, default=False, nullable=False)     joined_at = Column(DateTime(timezone=True), server_default=func.now())          # Обратные связи     user = relationship("User", back_populates="group_membership")     group = relationship("Group", back_populates="members")`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

> **Примечание:** Я добавил только основные модели для старта. Остальные (Events, Topics и т.д.) добавляются по аналогии.

**app/db/repository.py** (Функции для работы с базой данных)

      `from sqlalchemy.ext.asyncio import AsyncSession from sqlalchemy.future import select from sqlalchemy.orm import selectinload  from app.db.models import User, Group, GroupMember from uuid import UUID  class UserRepo:     def __init__(self, session: AsyncSession):         self.session = session      async def get_or_create_user(self, telegram_id: int, username: str, first_name: str, last_name: str | None) -> User:         stmt = select(User).where(User.telegram_id == telegram_id)         result = await self.session.execute(stmt)         user = result.scalar_one_or_none()          if not user:             user = User(                 telegram_id=telegram_id,                 telegram_username=username,                 first_name=first_name,                 last_name=last_name             )             self.session.add(user)             await self.session.commit()             await self.session.refresh(user)         return user      async def get_user_with_group_info(self, telegram_id: int) -> User | None:         """ Получает пользователя и информацию о его группе одним запросом """         stmt = (             select(User)             .options(selectinload(User.group_membership).selectinload(GroupMember.group))             .where(User.telegram_id == telegram_id)         )         result = await self.session.execute(stmt)         return result.scalar_one_or_none()  class GroupRepo:     def __init__(self, session: AsyncSession):         self.session = session          async def create_group(self, name: str, creator_id: int) -> Group:         """ Создает группу и делает создателя старостой """         # Создаем группу         new_group = Group(name=name, creator_id=creator_id)         self.session.add(new_group)         await self.session.flush() # Получаем group.id до коммита          # Добавляем создателя в группу как старосту         membership = GroupMember(             user_id=creator_id,             group_id=new_group.id,             is_leader=True         )         self.session.add(membership)         await self.session.commit()         await self.session.refresh(new_group)         return new_group`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

#### 4. Директория app/middlewares/

**app/middlewares/db.py** (Для передачи сессии БД в хэндлеры)

      `from typing import Callable, Dict, Any, Awaitable from aiogram import BaseMiddleware from aiogram.types import TelegramObject from sqlalchemy.ext.asyncio import async_sessionmaker  class DbSessionMiddleware(BaseMiddleware):     def __init__(self, session_pool: async_sessionmaker):         super().__init__()         self.session_pool = session_pool      async def __call__(         self,         handler: Callable[[TelegramObject, Dict[str, Any]], Awaitable[Any]],         event: TelegramObject,         data: Dict[str, Any],     ) -> Any:         async with self.session_pool() as session:             data["session"] = session             return await handler(event, data)`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

#### 5. Директория app/keyboards/

**app/keyboards/reply.py**

      `from aiogram.types import ReplyKeyboardMarkup, KeyboardButton  # Клавиатура для пользователя, который не состоит в группе main_menu_unregistered = ReplyKeyboardMarkup(     keyboard=[         [KeyboardButton(text="🚀 Создать группу")],         [KeyboardButton(text="🔗 Присоединиться по ссылке")]     ],     resize_keyboard=True )  # Клавиатура для старосты main_menu_leader = ReplyKeyboardMarkup(     keyboard=[         [KeyboardButton(text="📅 События и Бронь")],         [KeyboardButton(text="👥 Участники группы")],         [KeyboardButton(text="⚙️ Настройки группы")]     ],     resize_keyboard=True )  # ... Добавьте здесь другие клавиатуры по мере необходимости`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**app/keyboards/inline.py**

      `from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton  # Здесь будут ваши инлайн клавиатуры, например, для календаря или выбора тем # Пример: def get_example_inline_keyboard():     return InlineKeyboardMarkup(         inline_keyboard=[             [InlineKeyboardButton(text="Кнопка 1", callback_data="button1_pressed")],             [InlineKeyboardButton(text="Кнопка 2", callback_data="button2_pressed")]         ]     )`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

#### 6. Директория app/handlers/

**app/handlers/common.py** (Общие команды, регистрация)

      `from aiogram import Router, F from aiogram.types import Message from aiogram.filters import CommandStart from aiogram.fsm.context import FSMContext from sqlalchemy.ext.asyncio import AsyncSession  from app.db.repository import UserRepo from app.keyboards import reply from app.handlers.group_leader import CreateGroup  router = Router()  @router.message(CommandStart()) async def cmd_start(message: Message, session: AsyncSession):     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)      if not user: # Теоретически, этого не должно быть, но для надежности         user = await user_repo.get_or_create_user(             telegram_id=message.from_user.id,             username=message.from_user.username,             first_name=message.from_user.first_name,             last_name=message.from_user.last_name         )      # Проверяем, состоит ли пользователь в группе     if user.group_membership:         if user.group_membership.is_leader:             await message.answer(                 f"Добро пожаловать, староста группы «{user.group_membership.group.name}»!",                 reply_markup=reply.main_menu_leader             )         # ... здесь будут проверки на is_assistant и обычного участника         else:              await message.answer(f"Вы участник группы «{user.group_membership.group.name}»")     else:         await message.answer(             "👋 Добро пожаловать! Вы еще не состоите в группе.\n\nСоздайте свою или присоединитесь к существующей.",             reply_markup=reply.main_menu_unregistered         )  @router.message(F.text == "🚀 Создать группу") async def start_create_group(message: Message, state: FSMContext, session: AsyncSession):     user_repo = UserRepo(session)     user = await user_repo.get_user_with_group_info(message.from_user.id)     if user.group_membership:         await message.answer("Вы уже состоите в группе. Нельзя создать еще одну.")         return          await state.set_state(CreateGroup.waiting_for_name)     await message.answer("Отлично! Введите название для вашей новой группы:")`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**app/handlers/group_leader.py** (Функционал старосты)

      `from aiogram import Router, F from aiogram.types import Message from aiogram.fsm.state import StatesGroup, State from aiogram.fsm.context import FSMContext from sqlalchemy.ext.asyncio import AsyncSession  from app.db.repository import GroupRepo from app.keyboards import reply  router = Router()  class CreateGroup(StatesGroup):     waiting_for_name = State()  @router.message(CreateGroup.waiting_for_name) async def process_group_name(message: Message, state: FSMContext, session: AsyncSession):     group_name = message.text     if len(group_name) > 50:         await message.answer("Название слишком длинное. Пожалуйста, введите название до 50 символов.")         return      group_repo = GroupRepo(session)     await group_repo.create_group(name=group_name, creator_id=message.from_user.id)          await state.clear()     await message.answer(         f"🎉 Группа «{group_name}» успешно создана! Вы теперь её староста.",         reply_markup=reply.main_menu_leader     )  @router.message(F.text == "📅 События и Бронь") async def handle_events_and_booking(message: Message):     # Здесь будет логика для раздела "События и Бронь"     await message.answer("Раздел 'События и Бронь' в разработке.")  @router.message(F.text == "👥 Участники группы") async def handle_group_members(message: Message):     # Здесь будет логика для раздела "Участники группы"     await message.answer("Раздел 'Участники группы' в разработке.")  @router.message(F.text == "⚙️ Настройки группы") async def handle_group_settings(message: Message):     # Здесь будет логика для раздела "Настройки группы"     await message.answer("Раздел 'Настройки группы' в разработке.")`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**app/handlers/admin.py, group_assistant.py, group_member.py** (Заглушки)

      `# Содержимое для каждого из этих файлов будет одинаковым на старте: from aiogram import Router  router = Router()  # Здесь будет логика для соответствующей роли # Например, для admin.py: # @router.message(Command("admin_panel")) # async def show_admin_panel(...): #     ...`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

**app/utils/tools.py** (Пока пустой, для будущих утилит)

      `# Здесь могут быть функции, например: # - Генерация случайных токенов для приглашений # - Форматирование дат # - и т.д.`
    

IGNORE_WHEN_COPYING_START

Use code [with caution](https://support.google.com/legal/answer/13505487). Python

IGNORE_WHEN_COPYING_END

---

### Шаг 3: Как это запустить

1. **Установите PostgreSQL** на свой компьютер или используйте облачный сервис.
    
2. **Создайте базу данных** (например, с именем stud_bot_db).
    
3. **Выполните SQL-скрипт**, который я давал в **предыдущем ответе**, чтобы создать все таблицы в этой базе данных.
    
4. **Склонируйте/скопируйте все файлы** из этого ответа в правильные папки.
    
5. **Установите зависимости:** pip install -r requirements.txt.
    
6. **Заполните файл .env** вашим токеном бота и правильными данными для подключения к БД.
    
7. **Запустите бота:** python bot.py.
    

Теперь ваш бот должен запуститься. Напишите ему /start. Он должен вас поприветствовать и предложить создать группу. После создания группы он должен признать вас старостой и показать соответствующую клавиатуру.

Это прочный и расширяемый скелет. Теперь вы можете постепенно, файл за файлом, добавлять остальной функционал. Удачи, и, пожалуйста, не крадите меня. Я еще пригожусь.