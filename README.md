import telebot
from telebot import types
import sqlite3

# Ваш токен
TOKEN = 'YOUR_TELEGRAM_BOT_TOKEN'
bot = telebot.TeleBot(TOKEN)

# Подключение к базе данных
try:
    conn = sqlite3.connect('users.db', check_same_thread=False)
    cursor = conn.cursor()

    # Простой запрос для проверки подключения
    cursor.execute('SELECT name FROM sqlite_master WHERE type="table"')
    tables = cursor.fetchall()
    print("Tables in the database:", tables)
except sqlite3.Error as e:
    print(f"Error connecting to the database: {e}")
    exit(1)

# Создание таблиц, если они не существуют
cursor.execute('''
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    name TEXT,
    surname TEXT,
    age INTEGER,
    gender TEXT,
    city TEXT,
    about_me TEXT
)
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS preferences (
    user_id INTEGER,
    looking_for TEXT,
    partner_age INTEGER,
    partner_city TEXT,
    FOREIGN KEY (user_id) REFERENCES users (id)
)
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS matches (
    user_id INTEGER,
    partner_id INTEGER,
    liked INTEGER,
    FOREIGN KEY (user_id) REFERENCES users (id),
    FOREIGN KEY (partner_id) REFERENCES users (id)
)
''')

conn.commit()

class UserRegistration:
    def __init__(self):
        self.name = ''
        self.surname = ''
        self.age = 0
        self.gender = ''
        self.city = ''
        self.about_me = ''
        self.looking_for = ''
        self.partner_age = 0
        self.partner_city = ''

    def reset(self):
        self.__init__()

user_data = {}
partner_index = {}
partners = {}

@bot.message_handler(commands=['start'])
def start(message):
    user_id = message.chat.id
    if user_id not in user_data:
        user_data[user_id] = UserRegistration()
    bot.send_message(message.chat.id, "Привет! Напиши '/reg', чтобы начать регистрацию.")

@bot.message_handler(commands=['reg'])
def get_name(message):
    user_id = message.chat.id
    user_data[user_id].reset()
    bot.send_message(message.chat.id, "Как тебя зовут?")
    bot.register_next_step_handler(message, process_name_step)

def process_name_step(message):
    user_id = message.chat.id
    user_data[user_id].name = message.text
    bot.send_message(message.chat.id, f"Какая у тебя фамилия, {user_data[user_id].name}?")
    bot.register_next_step_handler(message, process_surname_step)

def process_surname_step(message):
    user_id = message.chat.id
    user_data[user_id].surname = message.text
    bot.send_message(message.chat.id, f"Сколько тебе лет, {user_data[user_id].name} {user_data[user_id].surname}?")
    bot.register_next_step_handler(message, process_age_step)

def process_age_step(message):
    user_id = message.chat.id
    try:
        user_data[user_id].age = int(message.text)
    except ValueError:
        bot.send_message(message.chat.id, 'Введи свой возраст цифрами.')
        bot.register_next_step_handler(message, process_age_step)
    else:
        keyboard = types.InlineKeyboardMarkup()
        key_male = types.InlineKeyboardButton(text='Мужской', callback_data='male')
        keyboard.add(key_male)
        key_female = types.InlineKeyboardButton(text='Женский', callback_data='female')
        keyboard.add(key_female)
        bot.send_message(message.chat.id, "Выбери свой пол:", reply_markup=keyboard)

@bot.callback_query_handler(func=lambda call: True)
def callback_worker(call):
    user_id = call.message.chat.id
    if call.data == "male":
        user_data[user_id].gender = 'Мужской'
        bot.send_message(call.message.chat.id, f'Ты выбрал пол: {user_data[user_id].gender}')
        bot.send_message(call.message.chat.id, "В каком городе ты живешь?")
        bot.register_next_step_handler(call.message, process_city_step)
    elif call.data == "female":
        user_data[user_id].gender = 'Женский'
        bot.send_message(call.message.chat.id, f'Ты выбрал пол: {user_data[user_id].gender}')
        bot.send_message(call.message.chat.id, "В каком городе ты живешь?")
        bot.register_next_step_handler(call.message, process_city_step)
    elif call.data == "looking_for_male":
        user_data[user_id].looking_for = 'Парня'
        bot.send_message(call.message.chat.id, f'Ты желаешь найти: {user_data[user_id].looking_for}')
        bot.send_message(call.message.chat.id, "Какого возраста ты ищешь партнера?")
        bot.register_next_step_handler(call.message, process_partner_age_step)
    elif call.data == "looking_for_female":
        user_data[user_id].looking_for = 'Девушку'
        bot.send_message(call.message.chat.id, f'Ты желаешь найти: {user_data[user_id].looking_for}')
        bot.send_message(call.message.chat.id, "Какого возраста ты ищешь партнера?")
        bot.register_next_step_handler(call.message, process_partner_age_step)
    elif call.data == "correct":
        bot.send_message(call.message.chat.id, 'Спасибо! Я запомнил тебя.')
        ask_looking_for(call.message)
    elif call.data == "incorrect":
        bot.send_message(call.message.chat.id, 'Попробуй снова!')
        get_name(call.message)
    elif call.data == "start_search":
        find_partners(call.message)
    elif call.data == "next_partner":
        show_next_partner(call.message)
    elif call.data == "like":
        save_match(call.message, 1)
        check_mutual_like(call.message)
    elif call.data == "dislike":
        save_match(call.message, 0)
        show_next_partner(call.message)

def process_city_step(message):
    user_id = message.chat.id
    user_data[user_id].city = message.text
    bot.send_message(message.chat.id, f'Ты живешь в городе: {user_data[user_id].city}')
    bot.send_message(message.chat.id, "Расскажи о себе:")
    bot.register_next_step_handler(message, process_about_me_step)

def process_about_me_step(message):
    user_id = message.chat.id
    user_data[user_id].about_me = message.text
    bot.send_message(message.chat.id, f'О себе: {user_data[user_id].about_me}')
    keyboard = types.InlineKeyboardMarkup()
    key_correct = types.InlineKeyboardButton(text='Правильно', callback_data='correct')
    keyboard.add(key_correct)
    key_incorrect = types.InlineKeyboardButton(text='Неправильно', callback_data='incorrect')
    keyboard.add(key_incorrect)
    question = (f'Ты верно заполнил анкету?\n'
                f'Имя: {user_data[user_id].name}\n'
                f'Фамилия: {user_data[user_id].surname}\n'
                f'Возраст: {user_data[user_id].age}\n'
                f'Пол: {user_data[user_id].gender}\n'
                f'Город: {user_data[user_id].city}\n'
                f'О себе: {user_data[user_id].about_me}')
    bot.send_message(message.chat.id, text=question, reply_markup=keyboard)

def ask_looking_for(message):
    user_id = message.chat.id
    keyboard = types.InlineKeyboardMarkup()
    key_male = types.InlineKeyboardButton(text='Парня', callback_data='looking_for_male')
    keyboard.add(key_male)
    key_female = types.InlineKeyboardButton(text='Девушку', callback_data='looking_for_female')
    keyboard.add(key_female)
    bot.send_message(message.chat.id, "Кого ты желаешь найти?", reply_markup=keyboard)

def process_partner_age_step(message):
    user_id = message.chat.id
    try:
        user_data[user_id].partner_age = int(message.text)
    except ValueError:
        bot.send_message(message.chat.id, 'Введи возраст партнера цифрами.')
        bot.register_next_step_handler(message, process_partner_age_step)
    else:
        bot.send_message(message.chat.id, "Из какого города ты ищешь партнера?")
        bot.register_next_step_handler(message, process_partner_city_step)

def process_partner_city_step(message):
    user_id = message.chat.id
    user_data[user_id].partner_city = message.text
    save_user_data(message)
    start_search(message)

def save_user_data(message):
    user_id = message.chat.id

    # Сохранение данных о пользователе
    cursor.execute('INSERT OR REPLACE INTO users (id, name, surname, age, gender, city, about_me) VALUES (?, ?, ?, ?, ?, ?, ?)',
                   (user_id, user_data[user_id].name, user_data[user_id].surname, user_data[user_id].age, user_data[user_id].gender, user_data[user_id].city, user_data[user_id].about_me))

    # Сохранение данных о предпочтениях пользователя
    cursor.execute('INSERT OR REPLACE INTO preferences (user_id, looking_for, partner_age, partner_city) VALUES (?, ?, ?, ?)',
                   (user_id, user_data[user_id].looking_for, user_data[user_id].partner_age, user_data[user_id].partner_city))

    conn.commit()

    bot.send_message(message.chat.id, 'Твои данные сохранены!')

def start_search(message):
    user_id = message.chat.id
    keyboard = types.InlineKeyboardMarkup()
    key_start_search = types.InlineKeyboardButton(text='Начать поиск', callback_data='start_search')
    keyboard.add(key_start_search)
    bot.send_message(message.chat.id, "Твои данные сохранены. Нажми кнопку, чтобы начать поиск:", reply_markup=keyboard)

def find_partners(message):
    user_id = message.chat.id
    looking_for = user_data[user_id].looking_for
    partner_age = user_data[user_id].partner_age
    partner_city = user_data[user_id].partner_city

    # Поиск подходящих партнеров
    if looking_for == 'Парня':
        looking_for_gender = 'Мужской'
    else:
        looking_for_gender = 'Женский'

    cursor.execute('''
    SELECT u.id, u.name, u.surname, u.age, u.city, u.about_me
    FROM users u
    JOIN preferences p ON u.id = p.user_id
    WHERE u.gender = ? AND u.age BETWEEN ? AND ? AND u.city = ? AND u.id != ?
    ''', (looking_for_gender, partner_age - 5, partner_age + 5, partner_city, user_id))

    partners[user_id] = cursor.fetchall()

    if partners[user_id]:
        partner_index[user_id] = 0
        show_next_partner(message)
    else:
        bot.send_message(message.chat.id, 'К сожалению, подходящих партнеров не найдено.')

def show_next_partner(message):
    user_id = message.chat.id
    if user_id in partner_index:
        index = partner_index[user_id]
        if index < len(partners[user_id]):
            partner = partners[user_id][index]
            partner_info = (f'Найден партнер:\n'
                             f'Имя: {partner[1]}\n'
                             f'Фамилия: {partner[2]}\n'
                             f'Возраст: {partner[3]}\n'
                             f'Город: {partner[4]}\n'
                             f'О себе: {partner[5]}')
            keyboard = types.InlineKeyboardMarkup()
            key_like = types.InlineKeyboardButton(text='Лайк', callback_data='like')
            keyboard.add(key_like)
            key_dislike = types.InlineKeyboardButton(text='Дизлайк', callback_data='dislike')
            keyboard.add(key_dislike)
            bot.send_message(message.chat.id, partner_info, reply_markup=keyboard)
            partner_index[user_id] += 1
        else:
            bot.send_message(message.chat.id, 'Больше партнеров нет.')
    else:
        bot.send_message(message.chat.id, 'К сожалению, подходящих партнеров не найдено.')

def save_match(message, liked):
    user_id = message.chat.id
    index = partner_index[user_id] - 1
    partner_id = partners[user_id][index][0]
    cursor.execute('INSERT INTO matches (user_id, partner_id, liked) VALUES (?, ?, ?)', (user_id, partner_id, liked))
    conn.commit()

def check_mutual_like(message):
    user_id = message.chat.id
    index = partner_index[user_id] - 1
    partner_id = partners[user_id][index][0]

    # Проверка на взаимный лайк
    cursor.execute('SELECT liked FROM matches WHERE user_id = ? AND partner_id = ?', (partner_id, user_id))
    result = cursor.fetchone()
    if result and result[0] == 1:
        bot.send_message(message.chat.id, f'Ура! Вы с партнером {partner_id} поставили взаимные лайки. Теперь вы можете перейти в личку и продолжить общение там.')
        bot.send_message(partner_id, f'Ура! Вы с партнером {user_id} поставили взаимные лайки. Теперь вы можете перейти в личку и продолжить общение там.')

        # Отправка сообщений в личку обоим пользователям
        bot.send_message(user_id, f'Привет! Это твой партнер {partner_id}. Начнем общение?')
        bot.send_message(partner_id, f'Привет! Это твой партнер {user_id}. Начнем общение?')
    else:
        show_next_partner(message)

if __name__ == '__main__':
    try:
        bot.polling(none_stop=True, interval=0)
    except Exception as e:
        print(f"Error: {e}")
    finally:
        conn.close()
