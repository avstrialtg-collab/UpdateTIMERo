# tg_timer.py
# Plugin for exteraGram that tracks time spent in the app and displays it as an overlay.

from typing import Any, List
import os
import shutil
import json

# Импортируем requests (он предустановлен в среде)
import requests

from android.os import Handler, Looper, SystemClock
from android.view import Gravity, View, WindowManager, MotionEvent
from android.widget import TextView
from android.graphics import Color, PixelFormat
from android.graphics.drawable import GradientDrawable
from android.content import Context
from android.view import WindowManager

from base_plugin import BasePlugin, AppEvent
from ui.settings import Switch, Header, Divider, Text
from android_utils import run_on_ui_thread, log, R
from client_utils import get_last_fragment, run_on_queue

from java import dynamic_proxy

# ---------- НАСТРОЙКИ ОБНОВЛЕНИЙ (ЗАМЕНИТЕ НА СВОИ) ----------
UPDATE_CHECK_URL = "https://raw.githubusercontent.com/your-username/your-repo/main/version.json"
PLUGIN_DOWNLOAD_URL = "https://raw.githubusercontent.com/your-username/your-repo/main/tg_timer.py"
# --------------------------------------------------------------

__id__ = "tg_timer"
__name__ = "TgTimer"
__description__ = "poshel Nafik."
__author__ = "enfarse"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.3.3"
__version__ = "1.0.1"          # Должна совпадать с версией в version.json


class TouchListener(dynamic_proxy(View.OnTouchListener)):
    """Прокси-класс для обработки касаний (перетаскивания)."""
    def __init__(self, callback):
        super().__init__()
        self.callback = callback

    def onTouch(self, view, event):
        return self.callback(view, event)


class OverlayTimer:
    """Manages the overlay view and timer logic."""

    def __init__(self, plugin):
        self.plugin = plugin
        self.total_seconds = 0
        self.base_time = 0
        self.base_seconds = 0
        self.running = False
        self.handler = Handler(Looper.getMainLooper())
        self.update_runnable = R(self._update_timer)
        self.text_view = None
        self.window_manager = None
        self.layout_params = None
        self.is_showing = False
        self.last_display_text = ""

    # ---------- Работа с overlay ----------

    def create_overlay(self):
        """Create and show the overlay view."""
        log(">>> create_overlay() called")
        context = self._get_context()
        if context is None:
            log("❌ Cannot create overlay: no context")
            return
        log(f"✅ Context obtained: {context}")

        self.text_view = TextView(context)
        self.text_view.setTextColor(Color.WHITE)
        self.text_view.setPadding(
            self._dp(12), self._dp(6), self._dp(12), self._dp(6)
        )
        self.text_view.setTextSize(14)

        drawable = GradientDrawable()
        drawable.setColor(Color.argb(180, 0, 0, 0))
        drawable.setCornerRadius(self._dp(16))
        self.text_view.setBackground(drawable)
        self.text_view.setGravity(Gravity.CENTER)

        self.text_view.setOnTouchListener(TouchListener(self._touch_listener))

        self.window_manager = context.getSystemService(Context.WINDOW_SERVICE)
        if self.window_manager is None:
            log("❌ Failed to get WindowManager")
            return

        self.layout_params = WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
            | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL,
            PixelFormat.TRANSLUCENT
        )
        self.layout_params.gravity = Gravity.TOP | Gravity.LEFT
        x = self.plugin.get_setting("overlay_x", 16)
        y = self.plugin.get_setting("overlay_y", 16)
        self.layout_params.x = self._dp(x)
        self.layout_params.y = self._dp(y)

        try:
            self.window_manager.addView(self.text_view, self.layout_params)
            self.is_showing = True
            self._update_text()
            log("✅ Overlay added successfully")
        except Exception as e:
            log(f"❌ Failed to add overlay: {e}")
            self.is_showing = False

    def remove_overlay(self):
        """Remove the overlay view."""
        if self.is_showing and self.window_manager is not None and self.text_view is not None:
            try:
                self.window_manager.removeView(self.text_view)
                self.is_showing = False
                log("Overlay removed")
            except Exception as e:
                log(f"Error removing overlay: {e}")

    # ---------- Таймер ----------

    def start_timer(self):
        if self.running:
            return
        self.running = True
        self.base_time = SystemClock.elapsedRealtime()
        self.base_seconds = self.total_seconds
        self._schedule_update(0)
        log("⏱️ Timer started")

    def stop_timer(self):
        if not self.running:
            return
        self.running = False
        self.handler.removeCallbacks(self.update_runnable)
        self.total_seconds = self._get_current_seconds()
        self.plugin.set_setting("total_seconds", self.total_seconds, reload_settings=False)
        log(f"⏹️ Timer stopped, saved {self.total_seconds}s")

    def toggle_pause(self, pause: bool):
        if pause:
            self.stop_timer()
        else:
            self.total_seconds = self.plugin.get_setting("total_seconds", 0)
            self.start_timer()

    def _get_current_seconds(self) -> int:
        if not self.running:
            return self.total_seconds
        elapsed = SystemClock.elapsedRealtime() - self.base_time
        return self.base_seconds + int(elapsed // 1000)

    def _schedule_update(self, delay_ms: int):
        if not self.running:
            return
        self.handler.postDelayed(self.update_runnable, delay_ms)

    def _update_timer(self):
        if not self.running:
            return
        self.total_seconds = self._get_current_seconds()
        self._update_text()
        now = SystemClock.elapsedRealtime()
        elapsed = now - self.base_time
        next_tick = 1000 - (elapsed % 1000)
        if next_tick < 50:
            next_tick += 1000
        self._schedule_update(next_tick)

    def _update_text(self):
        if self.text_view is None or not self.is_showing:
            return
        show_seconds = self.plugin.get_setting("show_seconds", True)
        new_text = self._format_time(self.total_seconds, show_seconds)
        if new_text == self.last_display_text:
            return
        self.last_display_text = new_text
        run_on_ui_thread(lambda: self.text_view.setText(new_text))

    @staticmethod
    def _format_time(total_sec: int, show_seconds: bool) -> str:
        hours = total_sec // 3600
        minutes = (total_sec % 3600) // 60
        seconds = total_sec % 60
        if show_seconds:
            return f"{hours:02d}:{minutes:02d}:{seconds:02d}"
        else:
            return f"{hours:02d}:{minutes:02d}"

    # ---------- Перетаскивание ----------

    def _touch_listener(self, view: View, event: MotionEvent) -> bool:
        if not self.is_showing or self.layout_params is None:
            return False

        action = event.getAction()
        if action == MotionEvent.ACTION_DOWN:
            self._drag_start_x = event.getRawX()
            self._drag_start_y = event.getRawY()
            self._drag_orig_x = self.layout_params.x
            self._drag_orig_y = self.layout_params.y
            return True
        elif action == MotionEvent.ACTION_MOVE:
            dx = event.getRawX() - self._drag_start_x
            dy = event.getRawY() - self._drag_start_y
            new_x = self._drag_orig_x + int(dx)
            new_y = self._drag_orig_y + int(dy)
            self.layout_params.x = new_x
            self.layout_params.y = new_y
            try:
                self.window_manager.updateViewLayout(self.text_view, self.layout_params)
            except Exception as e:
                log(f"Error updating overlay position: {e}")
            return True
        elif action == MotionEvent.ACTION_UP or action == MotionEvent.ACTION_CANCEL:
            x_dp = self._px_to_dp(self.layout_params.x)
            y_dp = self._px_to_dp(self.layout_params.y)
            self.plugin.set_setting("overlay_x", x_dp, reload_settings=False)
            self.plugin.set_setting("overlay_y", y_dp, reload_settings=False)
            return True
        return False

    # ---------- Вспомогательные методы ----------

    def _get_context(self):
        fragment = get_last_fragment()
        if fragment is not None:
            activity = fragment.getParentActivity()
            if activity is not None:
                log("Using context from fragment")
                return activity

        try:
            from org.telegram.messenger import LaunchActivity
            activity = LaunchActivity.getInstance()
            if activity is not None:
                log("Using context from LaunchActivity.getInstance()")
                return activity
        except Exception as e:
            log(f"LaunchActivity fallback failed: {e}")

        from org.telegram.messenger import ApplicationLoader
        log("⚠️ Using ApplicationContext (may fail for overlay)")
        return ApplicationLoader.applicationContext

    def _dp(self, px):
        from org.telegram.messenger import AndroidUtilities
        return AndroidUtilities.dp(px)

    def _px_to_dp(self, px):
        from org.telegram.messenger import AndroidUtilities
        return int(px / AndroidUtilities.density)


class TgTimerPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.overlay = OverlayTimer(self)
        self._update_checked = False

    def on_plugin_load(self):
        self.log("TgTimer plugin loaded")
        self.overlay.total_seconds = self.get_setting("total_seconds", 0)
        self.overlay.create_overlay()
        self.overlay.start_timer()

        # Запускаем проверку обновлений в фоне
        run_on_queue(self._check_for_updates)

    def on_plugin_unload(self):
        self.log("TgTimer plugin unloading")
        self.overlay.stop_timer()
        self.overlay.remove_overlay()

    def on_app_event(self, event_type: AppEvent):
        if event_type == AppEvent.PAUSE:
            self.log("App paused, stopping timer")
            self.overlay.stop_timer()
        elif event_type == AppEvent.RESUME:
            self.log("App resumed, starting timer")
            self.overlay.total_seconds = self.get_setting("total_seconds", 0)
            self.overlay.start_timer()

    def create_settings(self) -> List[Any]:
        return [
            Header(text="TgTimer Settings"),
            Switch(
                key="show_overlay",
                text="Show timer overlay",
                default=True,
                subtext="Display a floating timer on screen",
                icon="msg_info",
                on_change=self._on_show_overlay_change,
            ),
            Switch(
                key="show_seconds",
                text="Show seconds",
                default=True,
                subtext="Display seconds in the timer",
                icon="msg_time",
                on_change=self._on_show_seconds_change,
            ),
            Divider(),
            Text(
                text="Current time spent",
                subtext=self._get_current_time_str(),
                icon="msg_time",
            ),
            Text(
                text="Reset timer",
                icon="msg_delete",
                red=True,
                on_click=self._on_reset_click,
            ),
            # Новая кнопка для ручной проверки обновлений
            Divider(),
            Text(
                text="Check for updates",
                icon="msg_download",
                on_click=self._on_check_updates_click,
            ),
        ]

    def _on_show_overlay_change(self, new_value: bool):
        if new_value:
            self.overlay.create_overlay()
        else:
            self.overlay.remove_overlay()

    def _on_show_seconds_change(self, new_value: bool):
        self.overlay._update_text()

    def _on_reset_click(self, view: View):
        from ui.alert import AlertDialogBuilder
        fragment = get_last_fragment()
        if fragment is None:
            return
        activity = fragment.getParentActivity()
        if activity is None:
            return

        builder = AlertDialogBuilder(activity)
        builder.set_title("Reset Timer")
        builder.set_message("Are you sure you want to reset the timer to zero?")
        builder.set_positive_button("Reset", lambda b, w: self._do_reset())
        builder.set_negative_button("Cancel", lambda b, w: b.dismiss())
        builder.show()

    def _do_reset(self):
        self.overlay.stop_timer()
        self.overlay.total_seconds = 0
        self.set_setting("total_seconds", 0, reload_settings=False)
        self.overlay.start_timer()
        self.set_setting("__dummy", None, reload_settings=True)
        from ui.bulletin import BulletinHelper
        BulletinHelper.show_success("Timer reset to zero.")

    def _get_current_time_str(self) -> str:
        total = self.get_setting("total_seconds", 0)
        show_sec = self.get_setting("show_seconds", True)
        return OverlayTimer._format_time(total, show_sec)

    # ---------- НОВЫЕ МЕТОДЫ ДЛЯ ОБНОВЛЕНИЙ ----------

    def _on_check_updates_click(self, view: View):
        """Ручной запуск проверки обновлений через настройки."""
        from ui.bulletin import BulletinHelper
        BulletinHelper.show_info("Checking for updates...", get_last_fragment())
        run_on_queue(self._check_for_updates)

    def _check_for_updates(self):
        """Проверяет наличие новой версии и при необходимости обновляет плагин."""
        if self._update_checked:
            return
        self._update_checked = True

        try:
            self.log("Checking for updates...")
            resp = requests.get(UPDATE_CHECK_URL, timeout=10)
            if resp.status_code != 200:
                self.log(f"Update check failed: HTTP {resp.status_code}")
                return

            data = resp.json()
            remote_version = data.get("version")
            download_url = data.get("download_url", PLUGIN_DOWNLOAD_URL)
            if not remote_version:
                self.log("Remote version not found in JSON")
                return

            # Используем packaging.version для корректного сравнения (предустановлен)
            from packaging.version import parse as parse_version
            if parse_version(remote_version) <= parse_version(__version__):
                self.log("Plugin is up to date")
                run_on_ui_thread(lambda: self._notify_no_update())
                return

            self.log(f"New version available: {remote_version} (current: {__version__})")
            new_file_content = self._download_plugin(download_url)
            if new_file_content is None:
                self.log("Failed to download new plugin file")
                return

            current_file = __file__
            if not current_file.endswith(".py"):
                self.log("Current file is not a .py file, cannot update")
                return

            backup_file = current_file + ".bak"
            try:
                shutil.copy2(current_file, backup_file)
                with open(current_file, "w", encoding="utf-8") as f:
                    f.write(new_file_content)
                self.log("Plugin file updated successfully")
            except Exception as e:
                self.log(f"Failed to replace plugin file: {e}")
                return

            run_on_ui_thread(lambda: self._notify_update(remote_version))

        except Exception as e:
            self.log(f"Update check error: {e}")

    def _download_plugin(self, url: str):
        try:
            resp = requests.get(url, timeout=15)
            if resp.status_code == 200:
                return resp.text
            else:
                self.log(f"Download failed: {resp.status_code}")
                return None
        except Exception as e:
            self.log(f"Download exception: {e}")
            return None

    def _notify_update(self, new_version: str):
        from ui.bulletin import BulletinHelper
        BulletinHelper.show_info(
            f"Plugin updated to v{new_version}. Please restart Telegram to apply changes.",
            get_last_fragment()
        )

    def _notify_no_update(self):
        from ui.bulletin import BulletinHelper
        BulletinHelper.show_info("No updates available.", get_last_fragment())
