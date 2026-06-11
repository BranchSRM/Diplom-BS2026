Код дипломной работы
from PyQt5.QtWidgets import QInputDialog
import sys
import sqlite3
import os
from PyQt5.QtWidgets import (QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
                             QPushButton, QLabel, QLineEdit, QStackedWidget, QTableWidget,
                             QTableWidgetItem, QTreeWidget, QTreeWidgetItem, QMessageBox,
                             QDialog, QFormLayout, QTextEdit, QFileDialog, QComboBox,
                             QHeaderView, QSplitter, QListWidget, QListWidgetItem,
                             QAbstractItemView, QGroupBox, QScrollArea, QFrame)
from PyQt5.QtCore import Qt, QUrl
from PyQt5.QtGui import QFont, QPalette, QColor, QIcon, QDesktopServices, QPixmap

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
            FOREIGN KEY(parent_id) REFERENCES weapon_classes(id)
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
            FOREIGN KEY(class_id) REFERENCES weapon_classes(id),
            FOREIGN KEY(author_id) REFERENCES users(id)
        )
    ''')

    cursor.execute("SELECT COUNT(*) FROM users WHERE login='admin'")
    if cursor.fetchone()[0] == 0:
        cursor.execute("INSERT INTO users (login, password, role) VALUES ('admin','admin123','admin')")

    cursor.execute("SELECT COUNT(*) FROM weapon_classes")
    if cursor.fetchone()[0] == 0:
        cold_id = cursor.execute(
            "INSERT INTO weapon_classes (name, parent_id) VALUES ('Холодное оружие', NULL)").lastrowid
        short_blade = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Короткоклинковое', ?)",
                                     (cold_id,)).lastrowid
        long_blade = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Длинноклинковое', ?)",
                                    (cold_id,)).lastrowid
        polearm = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Древковое', ?)",
                                 (cold_id,)).lastrowid
        chopping = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Рубящее', ?)",
                                  (cold_id,)).lastrowid
        crushing = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Дробящее', ?)",
                                  (cold_id,)).lastrowid
        piercing = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Колющее', ?)",
                                  (cold_id,)).lastrowid

        fire_id = cursor.execute(
            "INSERT INTO weapon_classes (name, parent_id) VALUES ('Огнестрельное оружие', NULL)").lastrowid
        short_barrel = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Короткоствольное', ?)",
                                      (fire_id,)).lastrowid
        pistols = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Пистолеты', ?)",
                                 (short_barrel,)).lastrowid
        revolvers = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Револьверы', ?)",
                                   (short_barrel,)).lastrowid
        derringers = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Дерринджеры', ?)",
                                    (short_barrel,)).lastrowid
        smgs = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Пистолеты-пулемёты', ?)",
                              (short_barrel,)).lastrowid

        rifled = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Нарезное', ?)",
                                (fire_id,)).lastrowid
        rifles = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Винтовки', ?)",
                                (rifled,)).lastrowid
        assault = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Автоматы', ?)",
                                 (rifled,)).lastrowid
        machine_guns = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Пулемёты', ?)",
                                      (rifled,)).lastrowid
        sniper = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Снайперские винтовки', ?)",
                                (rifled,)).lastrowid
        semi_auto = cursor.execute(
            "INSERT INTO weapon_classes (name, parent_id) VALUES ('Полуавтоматические винтовки', ?)",
            (rifled,)).lastrowid

        smoothbore = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Гладкоствольное', ?)",
                                    (fire_id,)).lastrowid
        shotguns = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Ружья', ?)",
                                  (smoothbore,)).lastrowid
        shotguns2 = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Дробовики', ?)",
                                   (smoothbore,)).lastrowid

        explosive = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Взрывное', ?)",
                                   (fire_id,)).lastrowid
        grenades = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Гранаты', ?)",
                                  (explosive,)).lastrowid
        launchers = cursor.execute("INSERT INTO weapon_classes (name, parent_id) VALUES ('Гранатомёты', ?)",
                                   (explosive,)).lastrowid

    conn.commit()
    conn.close()


class LoginDialog(QDialog):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Авторизация / Регистрация")
        self.setFixedSize(400, 250)
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
    def __init__(self, article_data):
        super().__init__()
        self.setWindowTitle(article_data[1])
        self.setMinimumSize(600, 500)
        layout = QVBoxLayout()

        title_label = QLabel(f"<h1>{article_data[1]}</h1>")
        title_label.setWordWrap(True)
        layout.addWidget(title_label)

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


from PyQt5.QtWidgets import QInputDialog


class AdminPanel(QWidget):
    def __init__(self, user):
        super().__init__()
        self.user = user
        layout = QVBoxLayout()

        tabs = QPushButton("Просмотр таблиц БД")
        tabs.clicked.connect(self.show_tables)
        add_editor_btn = QPushButton("Добавить редактора")
        add_editor_btn.clicked.connect(self.add_editor)
        add_class_btn = QPushButton("Добавить класс оружия")
        add_class_btn.clicked.connect(self.add_weapon_class)

        layout.addWidget(tabs)
        layout.addWidget(add_editor_btn)
        layout.addWidget(add_class_btn)
        layout.addStretch()
        self.setLayout(layout)

    def add_editor(self):
        dialog = QDialog(self)
        dialog.setWindowTitle("Добавить редактора")
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

    def show_tables(self):
        dialog = QDialog(self)
        dialog.setWindowTitle("Таблицы базы данных")
        dialog.setMinimumSize(800, 600)
        layout = QVBoxLayout()

        tabs_widget = QComboBox()
        tabs_widget.addItems(["users", "weapon_classes", "articles"])
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

    def add_editor(self):
        login, ok = QInputDialog.getText(self, "Добавить редактора", "Логин:")
        if ok and login:
            password, ok = QInputDialog.getText(self, "Добавить редактора", "Пароль:")
            if ok and password:
                conn = sqlite3.connect(DB_NAME)
                cursor = conn.cursor()
                try:
                    cursor.execute("INSERT INTO users (login, password, role) VALUES (?,?,'editor')", (login, password))
                    conn.commit()
                    QMessageBox.information(self, "Успех", "Редактор добавлен")
                except sqlite3.IntegrityError:
                    QMessageBox.warning(self, "Ошибка", "Логин уже занят")
                finally:
                    conn.close()

    def add_weapon_class(self):
        dialog = QDialog(self)
        dialog.setWindowTitle("Добавить класс оружия")
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
                dialog.accept()
            except sqlite3.IntegrityError:
                QMessageBox.warning(dialog, "Ошибка", "Такой класс уже существует")
            finally:
                conn.close()

        ok_btn.clicked.connect(add_class)
        cancel_btn.clicked.connect(dialog.reject)
        dialog.exec_()


class EditorPanel(QWidget):
    def __init__(self, user):
        super().__init__()
        self.user = user
        layout = QVBoxLayout()

        add_btn = QPushButton("Добавить статью")
        add_btn.clicked.connect(self.add_article)
        edit_btn = QPushButton("Редактировать статью")
        edit_btn.clicked.connect(self.edit_article)
        delete_btn = QPushButton("Удалить статью")
        delete_btn.clicked.connect(self.delete_article)

        layout.addWidget(add_btn)
        layout.addWidget(edit_btn)
        layout.addWidget(delete_btn)
        layout.addStretch()
        self.setLayout(layout)

    def add_article(self):
        dialog = QDialog(self)
        dialog.setWindowTitle("Добавить статью")
        dialog.setMinimumSize(500, 400)
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

    def get_articles_list(self):
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT id, title FROM articles")
        articles = [f"{row[0]}: {row[1]}" for row in cursor.fetchall()]
        conn.close()
        return articles


class UserPanel(QWidget):
    def __init__(self, user):
        super().__init__()
        self.user = user
        layout = QHBoxLayout()

        splitter = QSplitter(Qt.Horizontal)

        self.tree = QTreeWidget()
        self.tree.setHeaderLabel("Классы оружия")
        self.build_tree()
        self.tree.itemClicked.connect(self.on_class_selected)

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

        splitter.addWidget(self.tree)
        splitter.addWidget(right_widget)
        splitter.setSizes([300, 500])

        layout.addWidget(splitter)
        self.setLayout(layout)
        self.current_class_id = None
        self.all_articles = []

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
        class_id = item.data(0, Qt.UserRole)
        self.current_class_id = class_id
        self.load_articles(class_id)

    def load_articles(self, class_id=None):
        self.articles_list.clear()
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        if class_id:
            cursor.execute('''SELECT a.id, a.title FROM articles a
                              JOIN weapon_classes wc ON a.class_id = wc.id
                              WHERE wc.id = ? OR wc.parent_id = ?
                              OR wc.parent_id IN (SELECT id FROM weapon_classes WHERE parent_id = ?)
                              OR wc.parent_id IN (SELECT id FROM weapon_classes WHERE parent_id IN
                              (SELECT id FROM weapon_classes WHERE parent_id = ?))''',
                           (class_id, class_id, class_id, class_id))
        else:
            cursor.execute("SELECT id, title FROM articles")
        self.all_articles = cursor.fetchall()
        conn.close()
        self.display_articles(self.all_articles)

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
            dialog = ArticleViewDialog(article)
            dialog.exec_()


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
            QTreeWidget { background-color: #FFFFFF; border: 1px solid #C4B599; }
            QListWidget { background-color: #FFFFFF; border: 1px solid #C4B599; }
            QTableWidget { background-color: #FFFFFF; border: 1px solid #C4B599; 
                           gridline-color: #D4C5A9; }
            QHeaderView::section { background-color: #D4C5A9; padding: 4px; border: 1px solid #B8A88A; }
            QLabel { color: #2C2C2C; }
            QGroupBox { border: 1px solid #C4B599; border-radius: 4px; margin-top: 8px; padding-top: 16px; }
            QGroupBox::title { subcontrol-origin: margin; left: 10px; padding: 0 5px; }
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

        self.stacked = QStackedWidget()

        if self.user["role"] == "admin":
            self.stacked.addWidget(AdminPanel(self.user))
        elif self.user["role"] == "editor":
            self.stacked.addWidget(EditorPanel(self.user))
        else:
            self.stacked.addWidget(UserPanel(self.user))

        main_layout.addWidget(self.stacked)
        central.setLayout(main_layout)

    def logout(self):
        self.close()
        new_window = MainWindow()
        new_window.show()
        self.deleteLater()
        global app
        app.activeWindow = new_window


class App(QApplication):
    def __init__(self, argv):
        super().__init__(argv)
        self.activeWindow = None


if __name__ == "__main__":
    init_db()
    app = App(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec_())
