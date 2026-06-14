код дипломной работы:
import sys
import sqlite3
from PyQt5.QtWidgets import (QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
                             QPushButton, QLabel, QLineEdit, QStackedWidget, QTableWidget,
                             QTableWidgetItem, QTreeWidget, QTreeWidgetItem, QMessageBox,
                             QDialog, QFormLayout, QTextEdit, QFileDialog, QComboBox,
                             QSplitter, QListWidget, QListWidgetItem, QInputDialog, QTabWidget)
from PyQt5.QtCore import Qt, QUrl
from PyQt5.QtGui import QDesktopServices, QPixmap

DB_NAME = "weapons_wiki.db"


def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()

    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            login TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL,
            role TEXT NOT NULL CHECK(role IN ('user','editor','admin'))
        )
    ''')

    cursor.execute('''
        CREATE TABLE IF NOT EXISTS weapon_classes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT UNIQUE NOT NULL,
            parent_id INTEGER,
            FOREIGN KEY(parent_id) REFERENCES weapon_classes(id) ON DELETE CASCADE
        )
    ''')

    cursor.execute('''
        CREATE TABLE IF NOT EXISTS articles (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            content TEXT,
            photo_path TEXT,
            reference_link TEXT,
            class_id INTEGER,
            author_id INTEGER,
            FOREIGN KEY(class_id) REFERENCES weapon_classes(id) ON DELETE SET NULL,
            FOREIGN KEY(author_id) REFERENCES users(id)
        )
    ''')

    cursor.execute('''
        CREATE TABLE IF NOT EXISTS favorites (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            article_id INTEGER NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE,
            FOREIGN KEY(article_id) REFERENCES articles(id) ON DELETE CASCADE,
            UNIQUE(user_id, article_id)
        )
    ''')

    cursor.execute("SELECT COUNT(*) FROM users WHERE login='admin'")
    if cursor.fetchone()[0] == 0:
        cursor.execute("INSERT INTO users (login, password, role) VALUES ('admin','admin123','admin')")

    cursor.execute("SELECT COUNT(*) FROM weapon_classes")
    if cursor.fetchone()[0] == 0:
        cold_id = cursor.execute(
            "INSERT INTO weapon_classes (name, parent_id) VALUES ('Холодное оружие', NULL)").lastrowid
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Короткоклинковое', ?)", (cold_id,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Длинноклинковое', ?)", (cold_id,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Древковое', ?)", (cold_id,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Рубящее', ?)", (cold_id,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Дробящее', ?)", (cold_id,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Колющее', ?)", (cold_id,))

        fire_id = cursor.execute(
            "INSERT INTO weapon_classes (name, parent_id) VALUES ('Огнестрельное оружие', NULL)").lastrowid
        short_barrel = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Короткоствольное', ?)",
                                      (fire_id,)).lastrowid
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Пистолеты', ?)", (short_barrel,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Револьверы', ?)", (short_barrel,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Дерринджеры', ?)", (short_barrel,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Пистолеты-пулемёты', ?)", (short_barrel,))

        rifled = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Нарезное', ?)",
                                (fire_id,)).lastrowid
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Винтовки', ?)", (rifled,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Автоматы', ?)", (rifled,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Пулемёты', ?)", (rifled,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Снайперские винтовки', ?)", (rifled,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Полуавтоматические винтовки', ?)",
                       (rifled,))

        smoothbore = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Гладкоствольное', ?)",
                                    (fire_id,)).lastrowid
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Ружья', ?)", (smoothbore,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Дробовики', ?)", (smoothbore,))

        explosive = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Взрывное', ?)",
                                   (fire_id,)).lastrowid
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Гранаты', ?)", (explosive,))
        cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Гранатомёты', ?)", (explosive,))

    conn.commit()
    conn.close()


class LoginDialog(QDialog):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Авторизация / Регистрация")
        self.setFixedSize(400, 250)
        self.setWindowFlags(self.windowFlags() & ~Qt.WindowContextHelpButtonHint)
        self.user = None

        layout = QVBoxLayout()

        self.stacked = QStackedWidget()

        auth_widget = QWidget()
        auth_layout = QFormLayout()
        self.auth_login = QLineEdit()
        self.auth_password = QLineEdit()
        self.auth_password.setEchoMode(QLineEdit.Password)
        auth_layout.addRow("Логин:", self.auth_login)
        auth_layout.addRow("Пароль:", self.auth_password)
        auth_btn = QPushButton("Войти")
        auth_btn.clicked.connect(self.login)
        auth_layout.addRow(auth_btn)
        auth_widget.setLayout(auth_layout)

        reg_widget = QWidget()
        reg_layout = QFormLayout()
        self.reg_login = QLineEdit()
        self.reg_password = QLineEdit()
        self.reg_password.setEchoMode(QLineEdit.Password)
        reg_layout.addRow("Логин:", self.reg_login)
        reg_layout.addRow("Пароль:", self.reg_password)
        reg_btn = QPushButton("Зарегистрироваться")
        reg_btn.clicked.connect(self.register)
        reg_layout.addRow(reg_btn)
        reg_widget.setLayout(reg_layout)

        self.stacked.addWidget(auth_widget)
        self.stacked.addWidget(reg_widget)

        switch_btn = QPushButton("Переключить на регистрацию")
        switch_btn.clicked.connect(self.switch_mode)

        layout.addWidget(self.stacked)
        layout.addWidget(switch_btn)
        self.setLayout(layout)
        self.mode = "auth"

    def switch_mode(self):
        if self.mode == "auth":
            self.mode = "reg"
            self.stacked.setCurrentIndex(1)
            self.sender().setText("Переключить на авторизацию")
        else:
            self.mode = "auth"
            self.stacked.setCurrentIndex(0)
            self.sender().setText("Переключить на регистрацию")

    def login(self):
        login = self.auth_login.text().strip()
        password = self.auth_password.text().strip()
        if not login or not password:
            QMessageBox.warning(self, "Ошибка", "Заполните все поля")
            return

        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT id, role FROM users WHERE login=? AND password=?", (login, password))
        user = cursor.fetchone()
        conn.close()

        if user:
            self.user = {"id": user[0], "login": login, "role": user[1]}
            self.accept()
        else:
            QMessageBox.warning(self, "Ошибка", "Неверный логин или пароль")

    def register(self):
        login = self.reg_login.text().strip()
        password = self.reg_password.text().strip()
        if not login or not password:
            QMessageBox.warning(self, "Ошибка", "Заполните все поля")
            return

        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        try:
            cursor.execute("INSERT INTO users (login, password, role) VALUES (?,?,'user')", (login, password))
            conn.commit()
            user_id = cursor.lastrowid
            self.user = {"id": user_id, "login": login, "role": "user"}
            conn.close()
            self.accept()
        except sqlite3.IntegrityError:
            QMessageBox.warning(self, "Ошибка", "Логин уже занят")
        finally:
            conn.close()


class ArticleViewDialog(QDialog):
    def __init__(self, article_data, user_id=None, favorites_callback=None):
        super().__init__()
        self.article_data = article_data
        self.user_id = user_id
        self.favorites_callback = favorites_callback
        self.setWindowTitle(article_data[1])
        self.setMinimumSize(600, 500)
        self.setWindowFlags(self.windowFlags() & ~Qt.WindowContextHelpButtonHint)
        layout = QVBoxLayout()

        # Заголовок с кнопкой избранного
        title_layout = QHBoxLayout()
        title_label = QLabel(f"<h1>{article_data[1]}</h1>")
        title_label.setWordWrap(True)
        title_layout.addWidget(title_label)

        # Кнопка избранного (звезда) только для обычных пользователей
        if user_id:
            self.fav_btn = QPushButton("☆")
            self.fav_btn.setFixedSize(40, 40)
            self.fav_btn.setStyleSheet("""
                QPushButton { 
                    font-size: 24px; 
                    background-color: transparent; 
                    border: 2px solid #D4C5A9;
                    border-radius: 20px;
                }
                QPushButton:hover { background-color: #FFF3CD; }
            """)
            self.check_favorite_status()
            self.fav_btn.clicked.connect(self.toggle_favorite)
            title_layout.addWidget(self.fav_btn)
        title_layout.addStretch()
        layout.addLayout(title_layout)

        if article_data[3]:
            pixmap = QPixmap(article_data[3])
            if not pixmap.isNull():
                img_label = QLabel()
                img_label.setPixmap(pixmap.scaledToWidth(400, Qt.SmoothTransformation))
                layout.addWidget(img_label)

        content_label = QLabel(article_data[2] if article_data[2] else "")
        content_label.setWordWrap(True)
        content_label.setOpenExternalLinks(True)
        layout.addWidget(content_label)

        if article_data[4]:
            link_btn = QPushButton("Открыть ссылку в браузере")
            link_btn.clicked.connect(lambda: QDesktopServices.openUrl(QUrl(article_data[4])))
            layout.addWidget(link_btn)

        close_btn = QPushButton("Закрыть")
        close_btn.clicked.connect(self.close)
        layout.addWidget(close_btn)
        self.setLayout(layout)

    def check_favorite_status(self):
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT id FROM favorites WHERE user_id=? AND article_id=?",
                       (self.user_id, self.article_data[0]))
        is_favorite = cursor.fetchone() is not None
        conn.close()
        self.fav_btn.setText("★" if is_favorite else "☆")
        self.fav_btn.setStyleSheet("""
            QPushButton { 
                font-size: 24px; 
                background-color: """ + ("#FFD700" if is_favorite else "transparent") + """;
                border: 2px solid #D4C5A9;
                border-radius: 20px;
            }
            QPushButton:hover { background-color: #FFF3CD; }
        """)

    def toggle_favorite(self):
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        if self.fav_btn.text() == "☆":
            cursor.execute("INSERT INTO favorites (user_id, article_id) VALUES (?, ?)",
                           (self.user_id, self.article_data[0]))
            conn.commit()
            QMessageBox.information(self, "Успех", "Статья добавлена в избранное")
        else:
            cursor.execute("DELETE FROM favorites WHERE user_id=? AND article_id=?",
                           (self.user_id, self.article_data[0]))
            conn.commit()
            QMessageBox.information(self, "Успех", "Статья удалена из избранного")
        conn.close()
        self.check_favorite_status()
        if self.favorites_callback:
            self.favorites_callback()


# Базовый класс для панели просмотра статей (общий для всех ролей)
class ArticlesViewPanel(QWidget):
    def __init__(self, user=None):
        super().__init__()
        self.user = user
        layout = QHBoxLayout()

        splitter = QSplitter(Qt.Horizontal)

        # Левая панель с классами
        left_widget = QWidget()
        left_layout = QVBoxLayout()

        self.tree = QTreeWidget()
        self.tree.setHeaderLabel("Классы оружия")
        self.build_tree()
        self.tree.itemClicked.connect(self.on_class_selected)
        left_layout.addWidget(self.tree)

        # Кнопка избранного для пользователей
        if user and user.get("role") == "user":
            self.fav_view_btn = QPushButton("★ Избранное")
            self.fav_view_btn.clicked.connect(self.show_favorites)
            left_layout.addWidget(self.fav_view_btn)

        left_widget.setLayout(left_layout)

        right_widget = QWidget()
        right_layout = QVBoxLayout()

        self.articles_list = QListWidget()
        self.articles_list.itemDoubleClicked.connect(self.on_article_double_clicked)

        self.search_edit = QLineEdit()
        self.search_edit.setPlaceholderText("Поиск статей...")
        self.search_edit.textChanged.connect(self.filter_articles)

        right_layout.addWidget(self.search_edit)
        right_layout.addWidget(self.articles_list)
        right_widget.setLayout(right_layout)

        splitter.addWidget(left_widget)
        splitter.addWidget(right_widget)
        splitter.setSizes([300, 500])

        layout.addWidget(splitter)
        self.setLayout(layout)
        self.current_class_id = None
        self.all_articles = []
        self.showing_favorites = False

    def build_tree(self):
        self.tree.clear()
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT id, name, parent_id FROM weapon_classes")
        classes = cursor.fetchall()
        conn.close()

        def add_items(parent_item, parent_id):
            for cls_id, name, p_id in classes:
                if p_id == parent_id:
                    item = QTreeWidgetItem(parent_item if parent_item else self.tree, [name])
                    item.setData(0, Qt.UserRole, cls_id)
                    add_items(item, cls_id)

        add_items(None, None)
        self.tree.expandAll()

    def on_class_selected(self, item, column):
        if self.showing_favorites:
            self.showing_favorites = False
        class_id = item.data(0, Qt.UserRole)
        self.current_class_id = class_id
        self.load_articles(class_id)

    def load_articles(self, class_id=None):
        self.articles_list.clear()
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        if class_id:
            cursor.execute('''SELECT DISTINCT a.id, a.title FROM articles a
                              JOIN weapon_classes wc ON a.class_id = wc.id
                              WHERE wc.id = ? OR wc.parent_id = ?''',
                           (class_id, class_id))
        else:
            cursor.execute("SELECT id, title FROM articles")
        self.all_articles = cursor.fetchall()
        conn.close()
        self.display_articles(self.all_articles)

    def show_favorites(self):
        self.showing_favorites = True
        self.current_class_id = None
        self.articles_list.clear()
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute('''SELECT a.id, a.title FROM articles a
                          JOIN favorites f ON a.id = f.article_id
                          WHERE f.user_id = ?''', (self.user["id"],))
        favorites = cursor.fetchall()
        conn.close()
        self.all_articles = favorites
        self.display_articles(favorites)
        QMessageBox.information(self, "Избранное", f"Найдено {len(favorites)} статей в избранном")

    def display_articles(self, articles):
        self.articles_list.clear()
        for art_id, title in articles:
            item = QListWidgetItem(title)
            item.setData(Qt.UserRole, art_id)
            self.articles_list.addItem(item)

    def filter_articles(self, text):
        if not text:
            self.display_articles(self.all_articles)
            return
        filtered = [(aid, title) for aid, title in self.all_articles if text.lower() in title.lower()]
        self.display_articles(filtered)

    def on_article_double_clicked(self, item):
        article_id = item.data(Qt.UserRole)
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM articles WHERE id=?", (article_id,))
        article = cursor.fetchone()
        conn.close()
        if article:
            user_id = self.user["id"] if self.user and self.user.get("role") == "user" else None
            dialog = ArticleViewDialog(article, user_id, self.refresh_favorites)
            dialog.exec_()

    def refresh_favorites(self):
        if self.showing_favorites:
            self.show_favorites()


class AdminPanel(QWidget):
    def __init__(self, user):
        super().__init__()
        self.user = user
        layout = QVBoxLayout()

        # Создаем вкладки для админа
        self.tab_widget = QTabWidget()

        # Вкладка с просмотром статей
        self.articles_view = ArticlesViewPanel(user)
        self.tab_widget.addTab(self.articles_view, "Просмотр статей")

        # Вкладка с админскими функциями
        admin_widget = QWidget()
        admin_layout = QVBoxLayout()

        tabs = QPushButton("Просмотр таблиц БД")
        tabs.clicked.connect(self.show_tables)
        add_editor_btn = QPushButton("Добавить редактора")
        add_editor_btn.clicked.connect(self.add_editor)
        delete_user_btn = QPushButton("Удалить пользователя/редактора")
        delete_user_btn.clicked.connect(self.delete_user)
        add_class_btn = QPushButton("Добавить класс оружия")
        add_class_btn.clicked.connect(self.add_weapon_class)
        delete_class_btn = QPushButton("Удалить класс оружия")
        delete_class_btn.clicked.connect(self.delete_weapon_class)

        admin_layout.addWidget(tabs)
        admin_layout.addWidget(add_editor_btn)
        admin_layout.addWidget(delete_user_btn)
        admin_layout.addWidget(add_class_btn)
        admin_layout.addWidget(delete_class_btn)
        admin_layout.addStretch()
        admin_widget.setLayout(admin_layout)
        self.tab_widget.addTab(admin_widget, "Управление")

        layout.addWidget(self.tab_widget)
        self.setLayout(layout)

    def add_editor(self):
        dialog = QDialog(self)
        dialog.setWindowTitle("Добавить редактора")
        dialog.setWindowFlags(dialog.windowFlags() & ~Qt.WindowContextHelpButtonHint)
        layout = QFormLayout()

        login_edit = QLineEdit()
        password_edit = QLineEdit()
        password_edit.setEchoMode(QLineEdit.Password)

        layout.addRow("Логин:", login_edit)
        layout.addRow("Пароль:", password_edit)

        buttons = QHBoxLayout()
        ok_btn = QPushButton("Добавить")
        cancel_btn = QPushButton("Отмена")
        buttons.addWidget(ok_btn)
        buttons.addWidget(cancel_btn)
        layout.addRow(buttons)

        dialog.setLayout(layout)

        def save_editor():
            login = login_edit.text().strip()
            password = password_edit.text().strip()
            if not login or not password:
                QMessageBox.warning(dialog, "Ошибка", "Заполните все поля")
                return

            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            try:
                cursor.execute("INSERT INTO users (login, password, role) VALUES (?,?,'editor')", (login, password))
                conn.commit()
                QMessageBox.information(dialog, "Успех", "Редактор добавлен")
                dialog.accept()
            except sqlite3.IntegrityError:
                QMessageBox.warning(dialog, "Ошибка", "Логин уже занят")
            finally:
                conn.close()

        ok_btn.clicked.connect(save_editor)
        cancel_btn.clicked.connect(dialog.reject)
        dialog.exec_()

    def delete_user(self):
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT id, login, role FROM users WHERE role IN ('user', 'editor')")
        users = cursor.fetchall()
        conn.close()

        if not users:
            QMessageBox.information(self, "Инфо", "Нет пользователей для удаления")
            return

        items = [f"{u[0]}: {u[1]} ({u[2]})" for u in users]
        item, ok = QInputDialog.getItem(self, "Удалить пользователя", "Выберите пользователя:", items, 0, False)

        if ok and item:
            user_id = int(item.split(":")[0])
            reply = QMessageBox.question(self, "Подтверждение",
                                         "Удалить этого пользователя? Все его статьи также будут удалены.",
                                         QMessageBox.Yes | QMessageBox.No)
            if reply == QMessageBox.Yes:
                conn = sqlite3.connect(DB_NAME)
                cursor = conn.cursor()
                cursor.execute("DELETE FROM users WHERE id=?", (user_id,))
                conn.commit()
                conn.close()
                QMessageBox.information(self, "Успех", "Пользователь удален")

    def show_tables(self):
        dialog = QDialog(self)
        dialog.setWindowTitle("Таблицы базы данных")
        dialog.setMinimumSize(800, 600)
        dialog.setWindowFlags(dialog.windowFlags() & ~Qt.WindowContextHelpButtonHint)
        layout = QVBoxLayout()

        tabs_widget = QComboBox()
        tabs_widget.addItems(["users", "weapon_classes", "articles", "favorites"])
        layout.addWidget(tabs_widget)

        table = QTableWidget()
        layout.addWidget(table)

        def load_table():
            table_name = tabs_widget.currentText()
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute(f"SELECT * FROM {table_name}")
            rows = cursor.fetchall()
            columns = [desc[0] for desc in cursor.description]
            conn.close()

            table.setColumnCount(len(columns))
            table.setHorizontalHeaderLabels(columns)
            table.setRowCount(len(rows))
            for i, row in enumerate(rows):
                for j, val in enumerate(row):
                    item = QTableWidgetItem(str(val) if val is not None else "")
                    table.setItem(i, j, item)
            table.resizeColumnsToContents()

        tabs_widget.currentIndexChanged.connect(load_table)
        load_table()

        dialog.setLayout(layout)
        dialog.exec_()

    def add_weapon_class(self):
        dialog = QDialog(self)
        dialog.setWindowTitle("Добавить класс оружия")
        dialog.setWindowFlags(dialog.windowFlags() & ~Qt.WindowContextHelpButtonHint)
        layout = QFormLayout()
        name_edit = QLineEdit()
        parent_combo = QComboBox()
        parent_combo.addItem("Нет (корневой)", None)

        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT id, name FROM weapon_classes")
        for cls_id, cls_name in cursor.fetchall():
            parent_combo.addItem(cls_name, cls_id)
        conn.close()

        layout.addRow("Название:", name_edit)
        layout.addRow("Родительский класс:", parent_combo)

        buttons = QHBoxLayout()
        ok_btn = QPushButton("Добавить")
        cancel_btn = QPushButton("Отмена")
        buttons.addWidget(ok_btn)
        buttons.addWidget(cancel_btn)
        layout.addRow(buttons)

        dialog.setLayout(layout)

        def add_class():
            name = name_edit.text().strip()
            if not name:
                QMessageBox.warning(dialog, "Ошибка", "Введите название")
                return
            parent_id = parent_combo.currentData()
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            try:
                cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES (?,?)", (name, parent_id))
                conn.commit()
                QMessageBox.information(dialog, "Успех", "Класс добавлен")
                self.articles_view.build_tree()  # Обновляем дерево классов
                dialog.accept()
            except sqlite3.IntegrityError:
                QMessageBox.warning(dialog, "Ошибка", "Такой класс уже существует")
            finally:
                conn.close()

        ok_btn.clicked.connect(add_class)
        cancel_btn.clicked.connect(dialog.reject)
        dialog.exec_()

    def delete_weapon_class(self):
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT id, name FROM weapon_classes")
        classes = cursor.fetchall()
        conn.close()

        if not classes:
            QMessageBox.information(self, "Инфо", "Нет классов для удаления")
            return

        items = [f"{c[0]}: {c[1]}" for c in classes]
        item, ok = QInputDialog.getItem(self, "Удалить класс", "Выберите класс для удаления:", items, 0, False)

        if ok and item:
            class_id = int(item.split(":")[0])
            reply = QMessageBox.question(self, "Подтверждение",
                                         "Удалить этот класс? Все вложенные классы и статьи также будут удалены.",
                                         QMessageBox.Yes | QMessageBox.No)
            if reply == QMessageBox.Yes:
                conn = sqlite3.connect(DB_NAME)
                cursor = conn.cursor()
                cursor.execute("DELETE FROM weapon_classes WHERE id=?", (class_id,))
                conn.commit()
                conn.close()
                QMessageBox.information(self, "Успех", "Класс удален")
                self.articles_view.build_tree()  # Обновляем дерево классов
                self.articles_view.load_articles()  # Обновляем список статей


class EditorPanel(QWidget):
    def __init__(self, user):
        super().__init__()
        self.user = user
        layout = QVBoxLayout()

        # Создаем вкладки для редактора
        self.tab_widget = QTabWidget()

        # Вкладка с просмотром статей
        self.articles_view = ArticlesViewPanel(user)
        self.tab_widget.addTab(self.articles_view, "Просмотр статей")

        # Вкладка с редактированием
        edit_widget = QWidget()
        edit_layout = QVBoxLayout()

        add_btn = QPushButton("Добавить статью")
        add_btn.clicked.connect(self.add_article)
        edit_btn = QPushButton("Редактировать статью")
        edit_btn.clicked.connect(self.edit_article)
        delete_btn = QPushButton("Удалить статью")
        delete_btn.clicked.connect(self.delete_article)

        edit_layout.addWidget(add_btn)
        edit_layout.addWidget(edit_btn)
        edit_layout.addWidget(delete_btn)
        edit_layout.addStretch()
        edit_widget.setLayout(edit_layout)
        self.tab_widget.addTab(edit_widget, "Редактирование")

        layout.addWidget(self.tab_widget)
        self.setLayout(layout)

    def add_article(self):
        dialog = QDialog(self)
        dialog.setWindowTitle("Добавить статью")
        dialog.setMinimumSize(500, 400)
        dialog.setWindowFlags(dialog.windowFlags() & ~Qt.WindowContextHelpButtonHint)
        layout = QFormLayout()

        title_edit = QLineEdit()
        content_edit = QTextEdit()
        photo_path_edit = QLineEdit()
        photo_btn = QPushButton("Выбрать фото")
        link_edit = QLineEdit()
        class_combo = QComboBox()

        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT id, name, parent_id FROM weapon_classes")
        classes = cursor.fetchall()
        conn.close()

        def populate_combo(items, parent_id=None, level=0):
            for cls_id, name, p_id in items:
                if p_id == parent_id:
                    prefix = "  " * level + "└ " if level > 0 else ""
                    class_combo.addItem(f"{prefix}{name}", cls_id)
                    populate_combo(items, cls_id, level + 1)

        populate_combo(classes)

        def select_photo():
            path, _ = QFileDialog.getOpenFileName(dialog, "Выбрать фото", "", "Images (*.png *.jpg *.jpeg *.bmp)")
            if path:
                photo_path_edit.setText(path)

        photo_btn.clicked.connect(select_photo)

        layout.addRow("Название:", title_edit)
        layout.addRow("Класс оружия:", class_combo)
        layout.addRow("Содержание:", content_edit)
        layout.addRow("Ссылка:", link_edit)
        layout.addRow("Фото:", photo_path_edit)
        layout.addRow("", photo_btn)

        buttons = QHBoxLayout()
        save_btn = QPushButton("Сохранить")
        cancel_btn = QPushButton("Отмена")
        buttons.addWidget(save_btn)
        buttons.addWidget(cancel_btn)
        layout.addRow(buttons)

        dialog.setLayout(layout)

        def save_article():
            title = title_edit.text().strip()
            if not title:
                QMessageBox.warning(dialog, "Ошибка", "Введите название")
                return
            content = content_edit.toHtml()
            photo = photo_path_edit.text().strip() or None
            link = link_edit.text().strip() or None
            class_id = class_combo.currentData()

            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute('''INSERT INTO articles (title, content, photo_path, reference_link, class_id, author_id)
                              VALUES (?,?,?,?,?,?)''',
                           (title, content, photo, link, class_id, self.user["id"]))
            conn.commit()
            conn.close()
            QMessageBox.information(dialog, "Успех", "Статья добавлена")
            dialog.accept()
            self.articles_view.load_articles(self.articles_view.current_class_id)

        save_btn.clicked.connect(save_article)
        cancel_btn.clicked.connect(dialog.reject)
        dialog.exec_()

    def edit_article(self):
        articles = self.get_articles_list()
        if not articles:
            QMessageBox.information(self, "Инфо", "Нет доступных статей")
            return

        item, ok = QInputDialog.getItem(self, "Выбрать статью", "Статья:", articles, 0, False)
        if ok and item:
            article_id = int(item.split(":")[0])
            dialog = QDialog(self)
            dialog.setWindowTitle("Редактировать статью")
            dialog.setMinimumSize(500, 400)
            dialog.setWindowFlags(dialog.windowFlags() & ~Qt.WindowContextHelpButtonHint)
            layout = QFormLayout()

            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute("SELECT title, content, photo_path, reference_link, class_id FROM articles WHERE id=?",
                           (article_id,))
            article = cursor.fetchone()
            conn.close()

            title_edit = QLineEdit(article[0])
            content_edit = QTextEdit()
            content_edit.setHtml(article[1] if article[1] else "")
            photo_path_edit = QLineEdit(article[2] if article[2] else "")
            link_edit = QLineEdit(article[3] if article[3] else "")
            class_combo = QComboBox()

            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute("SELECT id, name, parent_id FROM weapon_classes")
            classes = cursor.fetchall()
            conn.close()

            def populate_combo(items, parent_id=None, level=0):
                for cls_id, name, p_id in items:
                    if p_id == parent_id:
                        prefix = "  " * level + "└ " if level > 0 else ""
                        class_combo.addItem(f"{prefix}{name}", cls_id)
                        if cls_id == article[4]:
                            class_combo.setCurrentIndex(class_combo.count() - 1)
                        populate_combo(items, cls_id, level + 1)

            populate_combo(classes)

            def select_photo():
                path, _ = QFileDialog.getOpenFileName(dialog, "Выбрать фото", "", "Images (*.png *.jpg *.jpeg *.bmp)")
                if path:
                    photo_path_edit.setText(path)

            photo_btn = QPushButton("Выбрать фото")
            photo_btn.clicked.connect(select_photo)

            layout.addRow("Название:", title_edit)
            layout.addRow("Класс:", class_combo)
            layout.addRow("Содержание:", content_edit)
            layout.addRow("Ссылка:", link_edit)
            layout.addRow("Фото:", photo_path_edit)
            layout.addRow("", photo_btn)

            buttons = QHBoxLayout()
            save_btn = QPushButton("Сохранить")
            cancel_btn = QPushButton("Отмена")
            buttons.addWidget(save_btn)
            buttons.addWidget(cancel_btn)
            layout.addRow(buttons)

            dialog.setLayout(layout)

            def save_changes():
                title = title_edit.text().strip()
                if not title:
                    QMessageBox.warning(dialog, "Ошибка", "Введите название")
                    return
                conn = sqlite3.connect(DB_NAME)
                cursor = conn.cursor()
                cursor.execute('''UPDATE articles SET title=?, content=?, photo_path=?, reference_link=?, class_id=?
                                  WHERE id=?''',
                               (title, content_edit.toHtml(), photo_path_edit.text().strip() or None,
                                link_edit.text().strip() or None, class_combo.currentData(), article_id))
                conn.commit()
                conn.close()
                QMessageBox.information(dialog, "Успех", "Статья обновлена")
                dialog.accept()
                self.articles_view.load_articles(self.articles_view.current_class_id)

            save_btn.clicked.connect(save_changes)
            cancel_btn.clicked.connect(dialog.reject)
            dialog.exec_()

    def delete_article(self):
        articles = self.get_articles_list()
        if not articles:
            QMessageBox.information(self, "Инфо", "Нет доступных статей")
            return

        item, ok = QInputDialog.getItem(self, "Удалить статью", "Статья:", articles, 0, False)
        if ok and item:
            article_id = int(item.split(":")[0])
            reply = QMessageBox.question(self, "Подтверждение", "Удалить статью?",
                                         QMessageBox.Yes | QMessageBox.No)
            if reply == QMessageBox.Yes:
                conn = sqlite3.connect(DB_NAME)
                cursor = conn.cursor()
                cursor.execute("DELETE FROM articles WHERE id=?", (article_id,))
                conn.commit()
                conn.close()
                QMessageBox.information(self, "Успех", "Статья удалена")
                self.articles_view.load_articles(self.articles_view.current_class_id)

    def get_articles_list(self):
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT id, title FROM articles")
        articles = [f"{row[0]}: {row[1]}" for row in cursor.fetchall()]
        conn.close()
        return articles


class UserPanel(ArticlesViewPanel):
    def __init__(self, user):
        super().__init__(user)


class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Оружейная вики")
        self.setMinimumSize(1000, 700)
        self.user = None

        self.setStyleSheet("""
            QMainWindow { background-color: #F5F5F0; }
            QWidget { background-color: #FAFAF5; color: #2C2C2C; font-family: 'Segoe UI', Arial; }
            QPushButton { background-color: #D4C5A9; border: 1px solid #B8A88A; padding: 8px 16px; 
                          border-radius: 4px; font-weight: bold; color: #2C2C2C; }
            QPushButton:hover { background-color: #C4B599; }
            QPushButton:pressed { background-color: #B0A080; }
            QLineEdit, QTextEdit, QComboBox { background-color: #FFFFFF; border: 1px solid #C4B599; 
                                              padding: 4px; border-radius: 3px; }
            QTreeWidget, QListWidget, QTableWidget { 
                background-color: #FFFFFF; border: 1px solid #C4B599; 
                outline: none;
            }
            QTreeWidget::item:selected, QListWidget::item:selected {
                background-color: #D4C5A9;
            }
            QHeaderView::section { background-color: #D4C5A9; padding: 4px; border: 1px solid #B8A88A; }
            QLabel { color: #2C2C2C; }
            QTabWidget::pane { border: 1px solid #C4B599; background-color: #FAFAF5; }
            QTabBar::tab { background-color: #E8E0D0; padding: 6px 12px; margin-right: 2px; }
            QTabBar::tab:selected { background-color: #D4C5A9; }
            QTabBar::tab:hover { background-color: #C4B599; }
        """)

        self.login_dialog = LoginDialog()
        if self.login_dialog.exec_() != QDialog.Accepted:
            sys.exit()
        self.user = self.login_dialog.user

        central = QWidget()
        self.setCentralWidget(central)
        main_layout = QVBoxLayout()

        header = QHBoxLayout()
        user_label = QLabel(f"Пользователь: {self.user['login']} ({self.user['role']})")
        user_label.setStyleSheet("font-size: 14px; font-weight: bold;")
        logout_btn = QPushButton("Выйти")
        logout_btn.clicked.connect(self.logout)
        header.addWidget(user_label)
        header.addStretch()
        header.addWidget(logout_btn)
        main_layout.addLayout(header)

        if self.user["role"] == "admin":
            self.panel = AdminPanel(self.user)
        elif self.user["role"] == "editor":
            self.panel = EditorPanel(self.user)
        else:
            self.panel = UserPanel(self.user)

        main_layout.addWidget(self.panel)
        central.setLayout(main_layout)

    def logout(self):
        self.close()
        self.new_window = MainWindow()
        self.new_window.show()


if __name__ == "__main__":
    init_db()
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec_())
