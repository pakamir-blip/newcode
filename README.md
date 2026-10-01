"""
r79.py
======
Phone-native Kivy rewrite baseline after Tkinter mobile frame failure.

Scope of r55 only:
- keep the approved shell layout
- keep FILE menu working with the stable touch-open logic
- keep the same touch-open logic on MAIN rows
- keep MAIN option 1 as GPS by icon only
- keep 5 GPS submenu items
- keep the internal map panel path from GPS -> MY LOCATION
- keep the bottom status bar for runtime messages
- keep the working GPS backend from r39
- keep the approved shell layout and GPS backend
- remove the internal satellite map engine from the active code path
- keep only a lightweight in-app shell for GPS status, coordinates, probe, and UTM
- remove external GPS/map launch too
- keep the GPS/UTM shell only; no map engine is active
- keep only one MAIN row open at a time

FILE items in this stage:
- NEW
- OPEN
- SAVE
- SAVE AS
- IMPORT
- EXPORT
- EXIT

Not included yet:
- line tool
- measurement logic
- polygon labels
- status bar
- bottom dock
"""

import ast
import importlib.util
import json
import math
import os
import sys
import threading
import time
import traceback
import urllib.error
import urllib.request
import webbrowser

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
VENDOR_DIR = os.path.join(BASE_DIR, 'vendor_py')
FONT_VAZIR = os.path.join(BASE_DIR, 'fonts', 'Vazirmatn-Regular.ttf')
MAP_CACHE_DIR = os.path.join(BASE_DIR, 'map_cache')
MAP_RUNTIME_CONFIG_PATH = os.path.join(BASE_DIR, 'map_runtime_config.py')
DEFAULT_MAP_LAT = 35.6892
DEFAULT_MAP_LON = 51.3890


def load_map_runtime_config():
    data = {
        'neshan_web_api_key': '',
        'iran_satellite_tile_url': '',
        'custom_road_tile_url': '',
        'custom_road_tile_ext': 'png',
        'parcel_overlay_tile_url': '',
        'parcel_overlay_tile_ext': 'png',
        'parcel_overlay_opacity': 0.86,
    }
    if not os.path.exists(MAP_RUNTIME_CONFIG_PATH):
        return data
    try:
        spec = importlib.util.spec_from_file_location('_pystation_map_runtime_config', MAP_RUNTIME_CONFIG_PATH)
        if spec is None or spec.loader is None:
            return data
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)
        data['neshan_web_api_key'] = str(getattr(module, 'NESHAN_WEB_API_KEY', '') or '').strip()
        data['iran_satellite_tile_url'] = str(getattr(module, 'IRAN_SATELLITE_TILE_URL', '') or '').strip()
        data['custom_road_tile_url'] = str(getattr(module, 'CUSTOM_ROAD_TILE_URL', '') or '').strip()
        data['custom_road_tile_ext'] = str(getattr(module, 'CUSTOM_ROAD_TILE_EXT', 'png') or 'png').strip().lower()
        data['parcel_overlay_tile_url'] = str(getattr(module, 'PARCEL_OVERLAY_TILE_URL', '') or '').strip()
        data['parcel_overlay_tile_ext'] = str(getattr(module, 'PARCEL_OVERLAY_TILE_EXT', 'png') or 'png').strip().lower()
        try:
            data['parcel_overlay_opacity'] = float(getattr(module, 'PARCEL_OVERLAY_OPACITY', 0.86) or 0.86)
        except Exception:
            data['parcel_overlay_opacity'] = 0.86
    except Exception:
        return data
    return data


MAP_RUNTIME_CONFIG = load_map_runtime_config()
NESHAN_WEB_API_KEY = (MAP_RUNTIME_CONFIG.get('neshan_web_api_key') or os.environ.get('NESHAN_WEB_API_KEY', '')).strip()
IRAN_SATELLITE_TILE_URL = (MAP_RUNTIME_CONFIG.get('iran_satellite_tile_url') or os.environ.get('IRAN_SATELLITE_TILE_URL', '')).strip()
CUSTOM_ROAD_TILE_URL = (MAP_RUNTIME_CONFIG.get('custom_road_tile_url') or os.environ.get('CUSTOM_ROAD_TILE_URL', '')).strip()
CUSTOM_ROAD_TILE_EXT = (MAP_RUNTIME_CONFIG.get('custom_road_tile_ext') or os.environ.get('CUSTOM_ROAD_TILE_EXT', 'png')).strip().lower() or 'png'
PARCEL_OVERLAY_TILE_URL = (MAP_RUNTIME_CONFIG.get('parcel_overlay_tile_url') or os.environ.get('PARCEL_OVERLAY_TILE_URL', '')).strip()
PARCEL_OVERLAY_TILE_EXT = (MAP_RUNTIME_CONFIG.get('parcel_overlay_tile_ext') or os.environ.get('PARCEL_OVERLAY_TILE_EXT', 'png')).strip().lower() or 'png'
try:
    PARCEL_OVERLAY_OPACITY = float(MAP_RUNTIME_CONFIG.get('parcel_overlay_opacity', os.environ.get('PARCEL_OVERLAY_OPACITY', 0.86)) or 0.86)
except Exception:
    PARCEL_OVERLAY_OPACITY = 0.86
WEB_ASSETS_DIR = os.path.join(BASE_DIR, 'web_assets')
LEAFLET_JS_PATH = os.path.join(WEB_ASSETS_DIR, 'leaflet.js')
LEAFLET_CSS_PATH = os.path.join(WEB_ASSETS_DIR, 'leaflet.css')


def safe_read_text_file(path, default=''):
    try:
        with open(path, 'r', encoding='utf-8') as fh:
            return fh.read()
    except Exception:
        return default


def escape_inline_script(text):
    return str(text or '').replace('</script', r'<\/script')


def leaflet_css_block():
    css_text = safe_read_text_file(LEAFLET_CSS_PATH, '')
    if css_text.strip():
        return f'<style id="leaflet-inline-css">\n{css_text}\n</style>'
    return '<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">'


def leaflet_js_block():
    js_text = escape_inline_script(safe_read_text_file(LEAFLET_JS_PATH, ''))
    if js_text.strip():
        return f'<script>\n{js_text}\n</script>'
    return '<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>'


if os.path.isdir(VENDOR_DIR) and VENDOR_DIR not in sys.path:
    sys.path.insert(0, VENDOR_DIR)

try:
    import arabic_reshaper
    from bidi.algorithm import get_display
except Exception:
    arabic_reshaper = None
    get_display = None

from kivy.config import Config
Config.set('input', 'mouse', 'mouse,multitouch_on_demand')
Config.set('graphics', 'resizable', False)
Config.set('graphics', 'width', '720')
Config.set('graphics', 'height', '1200')

from kivy.app import App
from kivy.clock import Clock
from kivy.core.window import Window
from kivy.graphics import Color, Rectangle, RoundedRectangle, Triangle, Line, Ellipse
from kivy.metrics import dp
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.button import Button
from kivy.uix.dropdown import DropDown
from kivy.uix.floatlayout import FloatLayout
from kivy.uix.image import Image
from kivy.uix.label import Label
from kivy.uix.stencilview import StencilView
from kivy.uix.widget import Widget

try:
    from android.runnable import run_on_ui_thread
except Exception:
    def run_on_ui_thread(func):
        return func

try:
    from jnius import PythonJavaClass, java_method
except Exception:
    class PythonJavaClass(object):
        pass

    def java_method(_signature):
        def decorator(func):
            return func
        return decorator


BG_APP = (0.06, 0.07, 0.09, 1)
BG_TOP = (0.10, 0.13, 0.17, 1)
BG_RAIL = (0.09, 0.12, 0.16, 1)
BG_WORK = (0.11, 0.14, 0.18, 1)
BG_SLOT = (0.16, 0.20, 0.26, 1)
BG_SLOT_ALT = (0.20, 0.25, 0.32, 1)
BG_ACTIVE = (0.17, 0.52, 0.84, 1)
BG_MENU_ITEM = (0.18, 0.22, 0.28, 1)
BG_SUB_PANEL = (0.15, 0.19, 0.25, 0.98)
BG_SUB_SLOT = (0.20, 0.25, 0.32, 1)
BG_MAP_CARD = (0.14, 0.17, 0.22, 0.98)
BG_MAP_VIEW = (0.17, 0.21, 0.18, 1.0)
BG_STATUS = (0.08, 0.10, 0.13, 1.0)
FG_MAIN = (0.92, 0.96, 1.00, 1)
FG_DIM = (0.68, 0.76, 0.87, 1)
FG_SOFT = (0.56, 0.64, 0.73, 0.22)
LINE_ACCENT = (0.18, 0.65, 1.00, 1)
ICON_SAT = (0.78, 0.89, 1.00, 1.0)
ROAD_COLOR = (0.85, 0.88, 0.92, 0.90)
BOUNDARY_COLOR = (0.18, 0.85, 0.96, 0.92)
MYLOC_COLOR = (0.20, 0.78, 1.00, 1.0)
BLACK = (0.0, 0.0, 0.0, 1.0)

TOP_H = dp(48)
STATUS_H = dp(34)
RAIL_W = dp(76)
SLOT_W = dp(64)
SLOT_H = dp(40)
RAIL_PAD_X = dp(4)
RAIL_TOP_PAD = dp(8)
RAIL_GAP = dp(4)
TITLE_H = dp(20)
MENU_W = dp(150)
MENU_ITEM_H = dp(34)
SUB_SLOT_W = dp(42)
SUB_SLOT_H = dp(34)
SUBMENU_SPACING = dp(4)
SUBMENU_PAD_X = dp(4)
SUBMENU_H = SLOT_H
SUBMENU_W = (SUB_SLOT_W * 5) + (SUBMENU_SPACING * 4) + (SUBMENU_PAD_X * 2)
ROW_GAP_X = dp(8)
ROW_W = RAIL_W + ROW_GAP_X + SUBMENU_W
MAP_CLEAR_LEFT = ROW_W + dp(18)
STACK_LAYOUT_BREAKPOINT = ROW_W + dp(250)
MOBILE_COMPACT_BREAKPOINT = ROW_W + dp(250)
WORK_OVERLAY_LANE_H = SLOT_H + dp(10)
TILE_ZOOM = 19
GPS_PARCEL_ZOOM = 19
SAT_INTERNAL_DEFAULT_ZOOM = 20
TILE_MIN_ZOOM = 3
TILE_MAX_ZOOM = 20
TILE_GRID = 5
TILE_VIEW_DIVISOR = 4.0
MAP_STYLE_DEFS = {
    'satellite': {
        'label': 'SATELLITE',
        'base_name': 'esri_world_imagery',
        'base_url': 'https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
        'base_ext': 'jpg',
        'overlay_name': None,
        'overlay_url': None,
        'overlay_ext': None,
        'overlay_opacity': 0.0,
        'max_source_zoom': 19,
    },
    'road': {
        'label': 'ROAD',
        'base_name': 'esri_world_topo',
        'base_url': 'https://services.arcgisonline.com/arcgis/rest/services/World_Topo_Map/MapServer/tile/{z}/{y}/{x}',
        'base_ext': 'jpg',
        'overlay_name': None,
        'overlay_url': None,
        'overlay_ext': None,
        'overlay_opacity': 0.0,
        'max_source_zoom': 19,
    },
    'bound': {
        'label': 'SATELLITE + OVERLAY',
        'base_name': 'esri_world_imagery',
        'base_url': 'https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
        'base_ext': 'jpg',
        'overlay_name': 'esri_reference',
        'overlay_url': 'https://services.arcgisonline.com/arcgis/rest/services/Reference/World_Reference_Overlay/MapServer/tile/{z}/{y}/{x}',
        'overlay_ext': 'png',
        'overlay_opacity': 0.72,
        'max_source_zoom': 19,
    },
    'hybrid': {
        'label': 'SATELLITE + OVERLAY',
        'base_name': 'esri_world_imagery',
        'base_url': 'https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
        'base_ext': 'jpg',
        'overlay_name': 'esri_reference',
        'overlay_url': 'https://services.arcgisonline.com/arcgis/rest/services/Reference/World_Reference_Overlay/MapServer/tile/{z}/{y}/{x}',
        'overlay_ext': 'png',
        'overlay_opacity': 0.72,
        'max_source_zoom': 19,
    },
}
FILE_MENU_ITEMS = ('NEW', 'OPEN', 'SAVE', 'SAVE AS', 'IMPORT', 'EXPORT', 'EXIT')
GPS_SUBMENU_ITEMS = [
    {'text': '', 'icon_kind': 'my_location', 'action': 'my_location'},
    {'text': 'SAT', 'icon_kind': None, 'action': 'satellite'},
    {'text': 'ROAD', 'icon_kind': None, 'action': 'road'},
    {'text': 'BOUND', 'icon_kind': None, 'action': 'boundary'},
    {'text': 'PIN', 'icon_kind': None, 'action': 'pin'},
]


def submenu_specs_for_main(main_index):
    if main_index == 1:
        return GPS_SUBMENU_ITEMS
    return [
        {'text': f'{main_index}-{child_index}', 'icon_kind': None, 'action': None}
        for child_index in range(1, 6)
    ]


def ui_text(text):
    if arabic_reshaper and get_display and any('\u0600' <= ch <= '\u06FF' for ch in text):
        try:
            return get_display(arabic_reshaper.reshape(text))
        except Exception:
            return text
    return text


def ui_font(preferred_path):
    return preferred_path if os.path.exists(preferred_path) else None

def ensure_map_cache_dir():
    try:
        os.makedirs(MAP_CACHE_DIR, exist_ok=True)
    except Exception:
        pass


def clamp_lat(lat):
    return max(-85.05112878, min(85.05112878, float(lat)))


def wrap_tile_x(x, zoom):
    max_tile = 2 ** zoom
    return int(x) % max_tile


def clamp_tile_y(y, zoom):
    max_tile = (2 ** zoom) - 1
    return max(0, min(int(y), max_tile))


def latlon_to_tile_float(lat, lon, zoom):
    lat = clamp_lat(lat)
    lon = float(lon)
    n = 2.0 ** zoom
    x = (lon + 180.0) / 360.0 * n
    lat_rad = math.radians(lat)
    y = (1.0 - math.asinh(math.tan(lat_rad)) / math.pi) / 2.0 * n
    return x, y


def normalize_tile_ext(value, default='png'):
    value = str(value or default).strip().lower().lstrip('.')
    return value or default


def map_style_def(style_key):
    style_key = style_key if style_key in MAP_STYLE_DEFS else 'hybrid'
    style = dict(MAP_STYLE_DEFS[style_key])

    style['max_source_zoom'] = int(round(float(style.get('max_source_zoom', 19))))

    if style_key == 'road' and CUSTOM_ROAD_TILE_URL:
        style['label'] = 'ROAD | CUSTOM SERVICE'
        style['base_name'] = 'custom_road_service'
        style['base_url'] = CUSTOM_ROAD_TILE_URL
        style['base_ext'] = normalize_tile_ext(CUSTOM_ROAD_TILE_EXT, 'png')

    if style_key in ('satellite', 'bound', 'hybrid') and PARCEL_OVERLAY_TILE_URL:
        style['overlay_name'] = 'parcel_overlay_service'
        style['overlay_url'] = PARCEL_OVERLAY_TILE_URL
        style['overlay_ext'] = normalize_tile_ext(PARCEL_OVERLAY_TILE_EXT, 'png')
        style['overlay_opacity'] = max(0.0, min(1.0, float(PARCEL_OVERLAY_OPACITY)))
        if style_key == 'satellite':
            style['label'] = 'SATELLITE + PARCEL OVERLAY'
        else:
            style['label'] = 'SATELLITE + OFFICIAL PARCELS'

    return style


def map_tile_cache_path(layer_name, zoom, x, y, ext):
    ensure_map_cache_dir()
    safe_layer = ''.join(ch if ch.isalnum() or ch in ('_', '-') else '_' for ch in str(layer_name))
    return os.path.join(MAP_CACHE_DIR, f'{safe_layer}_z{zoom}_x{x}_y{y}.{ext}')


def tile_xyz_bounds_4326(z, x, y):
    north, west = tile_float_to_latlon(x, y, z)
    south, east = tile_float_to_latlon(x + 1, y + 1, z)
    return west, south, east, north


def lonlat_to_webmercator(lon, lat):
    lon = max(-180.0, min(180.0, float(lon)))
    lat = clamp_lat(lat)
    x = lon * 20037508.342789244 / 180.0
    y = math.log(math.tan((90.0 + lat) * math.pi / 360.0)) / (math.pi / 180.0)
    y = y * 20037508.342789244 / 180.0
    return x, y


def tile_xyz_bounds_3857(z, x, y):
    west, south, east, north = tile_xyz_bounds_4326(z, x, y)
    minx, miny = lonlat_to_webmercator(west, south)
    maxx, maxy = lonlat_to_webmercator(east, north)
    return minx, miny, maxx, maxy


def render_tile_request_url(template, z, x, y):
    template = str(template or '')
    if not template:
        return ''
    west, south, east, north = tile_xyz_bounds_4326(z, x, y)
    minx, miny, maxx, maxy = tile_xyz_bounds_3857(z, x, y)
    replacements = {
        '{z}': str(int(z)),
        '{x}': str(int(x)),
        '{y}': str(int(y)),
        '{bbox_4326}': f'{west:.12f},{south:.12f},{east:.12f},{north:.12f}',
        '{bbox_epsg4326}': f'{west:.12f},{south:.12f},{east:.12f},{north:.12f}',
        '{bbox_3857}': f'{minx:.3f},{miny:.3f},{maxx:.3f},{maxy:.3f}',
        '{bbox_epsg3857}': f'{minx:.3f},{miny:.3f},{maxx:.3f},{maxy:.3f}',
        '{minx}': f'{minx:.3f}',
        '{miny}': f'{miny:.3f}',
        '{maxx}': f'{maxx:.3f}',
        '{maxy}': f'{maxy:.3f}',
        '{west}': f'{west:.12f}',
        '{south}': f'{south:.12f}',
        '{east}': f'{east:.12f}',
        '{north}': f'{north:.12f}',
    }
    for key, value in replacements.items():
        template = template.replace(key, value)
    return template


def download_map_tile(url, cache_path):
    if cache_path and os.path.exists(cache_path) and os.path.getsize(cache_path) > 0:
        return cache_path, None
    try:
        req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
        with urllib.request.urlopen(req, timeout=14) as response:
            data = response.read()
        if not data:
            return None, 'empty response'
        tmp_path = cache_path + '.tmp'
        with open(tmp_path, 'wb') as fh:
            fh.write(data)
        os.replace(tmp_path, cache_path)
        return cache_path, None
    except Exception as exc:
        try:
            if cache_path and os.path.exists(cache_path) and os.path.getsize(cache_path) > 0:
                return cache_path, None
        except Exception:
            pass
        return None, f'{exc.__class__.__name__}: {exc}'


def tile_float_to_latlon(x, y, zoom):
    n = 2.0 ** zoom
    lon = (float(x) / n) * 360.0 - 180.0
    lat_rad = math.atan(math.sinh(math.pi * (1.0 - (2.0 * float(y) / n))))
    lat = math.degrees(lat_rad)
    return lat, lon


def latlon_to_world_px(lat, lon, zoom):
    lat = clamp_lat(lat)
    lon = float(lon)
    scale = 256.0 * (2 ** zoom)
    x = (lon + 180.0) / 360.0 * scale
    siny = math.sin(math.radians(lat))
    siny = max(-0.9999, min(0.9999, siny))
    y = (0.5 - math.log((1.0 + siny) / (1.0 - siny)) / (4.0 * math.pi)) * scale
    return x, y


def world_px_to_latlon(x, y, zoom):
    scale = 256.0 * (2 ** zoom)
    lon = (float(x) / scale) * 360.0 - 180.0
    n = math.pi - (2.0 * math.pi * float(y) / scale)
    lat = math.degrees(math.atan(math.sinh(n)))
    lat = clamp_lat(lat)
    lon = ((lon + 180.0) % 360.0) - 180.0
    return lat, lon


def utm_zone_number(lat, lon):
    lon = float(lon)
    lat = float(lat)
    zone = int((lon + 180.0) / 6.0) + 1
    if 56.0 <= lat < 64.0 and 3.0 <= lon < 12.0:
        zone = 32
    if 72.0 <= lat < 84.0:
        if 0.0 <= lon < 9.0:
            zone = 31
        elif 9.0 <= lon < 21.0:
            zone = 33
        elif 21.0 <= lon < 33.0:
            zone = 35
        elif 33.0 <= lon < 42.0:
            zone = 37
    return max(1, min(60, zone))


def utm_band_letter(lat):
    lat = float(lat)
    if lat < -80 or lat > 84:
        return 'Z'
    bands = 'CDEFGHJKLMNPQRSTUVWX'
    index = int((lat + 80) / 8)
    index = max(0, min(index, len(bands) - 1))
    return bands[index]


def latlon_to_utm(lat, lon):
    lat = float(lat)
    lon = float(lon)
    a = 6378137.0
    f = 1 / 298.257223563
    e2 = f * (2 - f)
    ep2 = e2 / (1 - e2)
    k0 = 0.9996

    zone = utm_zone_number(lat, lon)
    band = utm_band_letter(lat)
    hemisphere = 'N' if lat >= 0 else 'S'

    lat_rad = math.radians(lat)
    lon_rad = math.radians(lon)
    lon0 = math.radians((zone - 1) * 6 - 180 + 3)

    sin_lat = math.sin(lat_rad)
    cos_lat = math.cos(lat_rad)
    tan_lat = math.tan(lat_rad)

    n = a / math.sqrt(1 - e2 * sin_lat * sin_lat)
    t = tan_lat * tan_lat
    c = ep2 * cos_lat * cos_lat
    A = cos_lat * (lon_rad - lon0)

    M = a * (
        (1 - e2 / 4 - 3 * e2 * e2 / 64 - 5 * e2 ** 3 / 256) * lat_rad
        - (3 * e2 / 8 + 3 * e2 * e2 / 32 + 45 * e2 ** 3 / 1024) * math.sin(2 * lat_rad)
        + (15 * e2 * e2 / 256 + 45 * e2 ** 3 / 1024) * math.sin(4 * lat_rad)
        - (35 * e2 ** 3 / 3072) * math.sin(6 * lat_rad)
    )

    easting = k0 * n * (
        A
        + (1 - t + c) * A ** 3 / 6
        + (5 - 18 * t + t * t + 72 * c - 58 * ep2) * A ** 5 / 120
    ) + 500000.0

    northing = k0 * (
        M
        + n * tan_lat * (
            A * A / 2
            + (5 - t + 9 * c + 4 * c * c) * A ** 4 / 24
            + (61 - 58 * t + t * t + 600 * c - 330 * ep2) * A ** 6 / 720
        )
    )
    if lat < 0:
        northing += 10000000.0

    return {
        'zone': zone,
        'band': band,
        'hemisphere': hemisphere,
        'easting': easting,
        'northing': northing,
    }


def choose_grid_spacing(meters_per_pixel):
    candidates = [10, 20, 50, 100, 200, 500, 1000, 2000, 5000, 10000]
    if meters_per_pixel <= 0:
        return 100
    target_px = 88.0
    for spacing in candidates:
        if (spacing / meters_per_pixel) >= target_px:
            return spacing
    return candidates[-1]


def format_utm_meters(value):
    try:
        return f"{int(round(float(value))):,}".replace(',', ' ')
    except Exception:
        return '--'


def build_leaflet_map_html(lat, lon, zoom, style_key='satellite'):
    style_key = style_key if style_key in MAP_STYLE_DEFS else 'satellite'
    html = """<!doctype html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=5.0, user-scalable=yes">
__LEAFLET_CSS_BLOCK__
<style>
html, body, #map { margin: 0; padding: 0; width: 100%; height: 100%; background: #17202a; overflow: hidden; }
.leaflet-control-container .leaflet-top.leaflet-left { top: 52px; }
#crosshair { position: fixed; left: 50%; top: 50%; width: 26px; height: 26px; margin-left: -13px; margin-top: -13px; pointer-events: none; z-index: 900; }
#crosshair:before, #crosshair:after { content: ''; position: absolute; background: rgba(36,177,255,.95); }
#crosshair:before { left: 12px; top: 0; width: 2px; height: 26px; }
#crosshair:after { left: 0; top: 12px; width: 26px; height: 2px; }
#cross-ring { position: absolute; left: 4px; top: 4px; width: 18px; height: 18px; border: 2px solid rgba(36,177,255,.95); border-radius: 999px; box-shadow: 0 0 0 1px rgba(0,0,0,.45); }
.box { position: fixed; z-index: 950; color: #eef6ff; background: rgba(11,18,27,.82); border: 1px solid rgba(56,154,255,.35); border-radius: 11px; padding: 7px 10px; font-family: sans-serif; line-height: 1.3; box-shadow: 0 6px 18px rgba(0,0,0,.25); }
#styleBox { left: 10px; top: 10px; font-size: 12px; font-weight: 700; }
#runtimeBox { left: 10px; top: 46px; right: 10px; font-size: 11px; color: #dff1ff; }
#runtimeBox.bad { color: #ffd4d4; border-color: rgba(255,95,95,.45); background: rgba(40,10,10,.86); }
#coordBox { left: 10px; bottom: 10px; right: 10px; font-size: 11px; }
#hintBox { right: 10px; top: 10px; font-size: 11px; text-align: right; }
</style>
</head>
<body>
<div id="map"></div>
<div id="crosshair"><div id="cross-ring"></div></div>
<div id="styleBox" class="box">MAP</div>
<div id="runtimeBox" class="box">ENGINE: starting...</div>
<div id="hintBox" class="box">1 finger: pan<br>2 fingers: pinch zoom</div>
<div id="coordBox" class="box">loading...</div>
__LEAFLET_JS_BLOCK__
<script>
const START_LAT = __LAT__;
const START_LON = __LON__;
const START_ZOOM = __ZOOM__;
const START_STYLE = "__STYLE__";
const styles = {
  satellite: {
    label: 'SATELLITE',
    base: 'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
    overlay: null,
    overlayOpacity: 0.0,
    attribution: 'Esri'
  },
  road: {
    label: 'ROAD / TOPO',
    base: 'https://services.arcgisonline.com/ArcGIS/rest/services/World_Topo_Map/MapServer/tile/{z}/{y}/{x}',
    overlay: null,
    overlayOpacity: 0.0,
    attribution: 'Esri'
  },
  bound: {
    label: 'SATELLITE + REFERENCE',
    base: 'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
    overlay: 'https://services.arcgisonline.com/arcgis/rest/services/Reference/World_Reference_Overlay/MapServer/tile/{z}/{y}/{x}',
    overlayOpacity: 0.75,
    attribution: 'Esri'
  },
  hybrid: {
    label: 'SATELLITE + REFERENCE',
    base: 'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
    overlay: 'https://services.arcgisonline.com/arcgis/rest/services/Reference/World_Reference_Overlay/MapServer/tile/{z}/{y}/{x}',
    overlayOpacity: 0.75,
    attribution: 'Esri'
  }
};
let currentBase = null;
let currentOverlay = null;
let myMarker = null;
let tileStartCount = 0;
let tileLoadCount = 0;
let tileErrorCount = 0;
let tileWatchdog = null;
function setRuntimeBox(text, isBad) {
  const box = document.getElementById('runtimeBox');
  box.textContent = text;
  if (isBad) box.classList.add('bad');
  else box.classList.remove('bad');
}
window.addEventListener('error', function(ev) {
  const msg = (ev && (ev.message || ev.error)) ? String(ev.message || ev.error) : 'unknown javascript error';
  setRuntimeBox('JS ERROR: ' + msg, true);
});
function bindLayerDiagnostics(layer, label) {
  if (!layer || !layer.on) return;
  layer.on('loading', function() {
    setRuntimeBox(label + ': requesting tiles...', false);
  });
  layer.on('tileloadstart', function() {
    tileStartCount += 1;
    if (tileStartCount <= 3) setRuntimeBox(label + ': loading tiles...', false);
  });
  layer.on('tileload', function() {
    tileLoadCount += 1;
    if (tileLoadCount === 1) setRuntimeBox(label + ': first tile received', false);
  });
  layer.on('load', function() {
    setRuntimeBox(label + ': tiles ready', false);
  });
  layer.on('tileerror', function() {
    tileErrorCount += 1;
    if (tileErrorCount >= 2) setRuntimeBox(label + ': tile source blocked or unreachable', true);
  });
}
const map = L.map('map', {
  center: [START_LAT, START_LON],
  zoom: START_ZOOM,
  zoomControl: true,
  attributionControl: true,
  dragging: true,
  inertia: true,
  inertiaDeceleration: 2500,
  easeLinearity: 0.20,
  worldCopyJump: true,
  touchZoom: true,
  doubleClickZoom: true,
  boxZoom: false,
  keyboard: false,
  zoomSnap: 0.25,
  zoomDelta: 0.25,
  tap: true,
  preferCanvas: true
});
L.control.scale({imperial:false}).addTo(map);
function utmZone(lat, lon) {
  let zone = Math.floor((lon + 180) / 6) + 1;
  if (lat >= 56 && lat < 64 && lon >= 3 && lon < 12) zone = 32;
  if (lat >= 72 && lat < 84) {
    if (lon >= 0 && lon < 9) zone = 31;
    else if (lon < 21) zone = 33;
    else if (lon < 33) zone = 35;
    else if (lon < 42) zone = 37;
  }
  return Math.max(1, Math.min(60, zone));
}
function utmBand(lat) {
  if (lat < -80 || lat > 84) return 'Z';
  const bands = 'CDEFGHJKLMNPQRSTUVWX';
  let idx = Math.floor((lat + 80) / 8);
  idx = Math.max(0, Math.min(idx, bands.length - 1));
  return bands[idx];
}
function latLonToUtm(lat, lon) {
  const a = 6378137.0;
  const f = 1 / 298.257223563;
  const e2 = f * (2 - f);
  const ep2 = e2 / (1 - e2);
  const k0 = 0.9996;
  const zone = utmZone(lat, lon);
  const band = utmBand(lat);
  const hemisphere = lat >= 0 ? 'N' : 'S';
  const latRad = lat * Math.PI / 180.0;
  const lonRad = lon * Math.PI / 180.0;
  const lon0 = ((zone - 1) * 6 - 180 + 3) * Math.PI / 180.0;
  const sinLat = Math.sin(latRad);
  const cosLat = Math.cos(latRad);
  const tanLat = Math.tan(latRad);
  const n = a / Math.sqrt(1 - e2 * sinLat * sinLat);
  const t = tanLat * tanLat;
  const c = ep2 * cosLat * cosLat;
  const A = cosLat * (lonRad - lon0);
  const M = a * (
    (1 - e2 / 4 - 3 * e2 * e2 / 64 - 5 * Math.pow(e2, 3) / 256) * latRad
    - (3 * e2 / 8 + 3 * e2 * e2 / 32 + 45 * Math.pow(e2, 3) / 1024) * Math.sin(2 * latRad)
    + (15 * e2 * e2 / 256 + 45 * Math.pow(e2, 3) / 1024) * Math.sin(4 * latRad)
    - (35 * Math.pow(e2, 3) / 3072) * Math.sin(6 * latRad)
  );
  let easting = k0 * n * (A + (1 - t + c) * Math.pow(A, 3) / 6 + (5 - 18 * t + t * t + 72 * c - 58 * ep2) * Math.pow(A, 5) / 120) + 500000.0;
  let northing = k0 * (M + n * tanLat * (A * A / 2 + (5 - t + 9 * c + 4 * c * c) * Math.pow(A, 4) / 24 + (61 - 58 * t + t * t + 600 * c - 330 * ep2) * Math.pow(A, 6) / 720));
  if (lat < 0) northing += 10000000.0;
  return {zone, band, hemisphere, easting, northing};
}
function applyStyle(key) {
  const style = styles[key] || styles.satellite;
  tileStartCount = 0;
  tileLoadCount = 0;
  tileErrorCount = 0;
  if (tileWatchdog) window.clearTimeout(tileWatchdog);
  if (currentBase) map.removeLayer(currentBase);
  if (currentOverlay) map.removeLayer(currentOverlay);
  currentBase = L.tileLayer(style.base, {maxZoom: 22, crossOrigin: true, updateWhenIdle: false, keepBuffer: 6, attribution: style.attribution});
  bindLayerDiagnostics(currentBase, style.label);
  currentBase.addTo(map);
  currentOverlay = null;
  if (style.overlay) {
    currentOverlay = L.tileLayer(style.overlay, {maxZoom: 22, crossOrigin: true, opacity: style.overlayOpacity, updateWhenIdle: false, keepBuffer: 6, attribution: style.attribution});
    bindLayerDiagnostics(currentOverlay, style.label + ' overlay');
    currentOverlay.addTo(map);
  }
  document.getElementById('styleBox').textContent = style.label;
  setRuntimeBox(style.label + ': engine ready', false);
  tileWatchdog = window.setTimeout(function() {
    if (tileLoadCount === 0) setRuntimeBox(style.label + ': no tile reached the WebView yet', true);
  }, 8000);
}
function updateInfo() {
  const c = map.getCenter();
  const u = latLonToUtm(c.lat, c.lng);
  const z = map.getZoom();
  document.getElementById('coordBox').innerHTML =
    'LAT: ' + c.lat.toFixed(6) + ' &nbsp; LON: ' + c.lng.toFixed(6) + '<br>' +
    'UTM: Z' + u.zone + u.band + u.hemisphere + '  E=' + u.easting.toFixed(2) + '  N=' + u.northing.toFixed(2) + '  ZOOM=' + z.toFixed(2);
}
myMarker = L.circleMarker([START_LAT, START_LON], {radius: 7, color: '#24b1ff', fillColor: '#24b1ff', fillOpacity: 1.0, weight: 2}).addTo(map);
applyStyle(START_STYLE);
updateInfo();
map.on('move zoom zoomend moveend', updateInfo);
window.setMapStyle = function(key) { applyStyle(key); };
window.setMyLocation = function(lat, lon, zoom) {
  myMarker.setLatLng([lat, lon]);
  map.setView([lat, lon], zoom || map.getZoom(), {animate: false});
  updateInfo();
};
window.setMapZoom = function(zoom) { map.setZoom(zoom); updateInfo(); };
</script>
</body>
</html>"""
    html = html.replace('__LAT__', f'{float(lat):.8f}')
    html = html.replace('__LON__', f'{float(lon):.8f}')
    html = html.replace('__ZOOM__', f'{float(zoom):.2f}')
    html = html.replace('__STYLE__', style_key)
    html = html.replace('__LEAFLET_CSS_BLOCK__', leaflet_css_block())
    html = html.replace('__LEAFLET_JS_BLOCK__', leaflet_js_block())
    return html


def build_leaflet_custom_satellite_html(lat, lon, zoom, tile_url, label='SATELLITE | CUSTOM TILE'):
    html = build_leaflet_map_html(lat, lon, zoom, 'satellite')
    html = html.replace(
        "label: 'SATELLITE'",
        f"label: {json.dumps(str(label))}",
        1,
    )
    html = html.replace(
        "base: 'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}'",
        f"base: {json.dumps(str(tile_url))}",
        1,
    )
    return html


def map_provider_runtime(style_key='road'):
    style_key = 'satellite' if style_key == 'satellite' else 'road'
    return {
        'provider': 'NESHAN',
        'style_key': style_key,
        'road_ready': bool(NESHAN_WEB_API_KEY),
        'satellite_ready': bool(IRAN_SATELLITE_TILE_URL),
        'api_key_present': bool(NESHAN_WEB_API_KEY),
        'satellite_tile_url_present': bool(IRAN_SATELLITE_TILE_URL),
        'config_loaded': bool(MAP_RUNTIME_CONFIG.get('neshan_web_api_key') or MAP_RUNTIME_CONFIG.get('iran_satellite_tile_url')),
    }


def build_provider_map_html(lat, lon, zoom, style_key='road'):
    style_key = 'satellite' if style_key == 'satellite' else 'road'
    runtime = map_provider_runtime(style_key)
    style_label = 'ROAD | NESHAN' if style_key == 'road' else 'SATELLITE | IRAN TILE'
    api_key_json = json.dumps(NESHAN_WEB_API_KEY)
    sat_tile_json = json.dumps(IRAN_SATELLITE_TILE_URL)

    if not runtime['api_key_present']:
        return f"""<!doctype html>
<html>
<head>
<meta charset=\"utf-8\">
<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no\">
<style>
html, body {{ margin: 0; padding: 0; width: 100%; height: 100%; background: #0d1117; color: #eef6ff; font-family: sans-serif; }}
body {{ display: flex; align-items: center; justify-content: center; }}
.card {{ max-width: 92%; border: 1px solid rgba(56,154,255,.45); background: rgba(17,23,31,.96); border-radius: 14px; padding: 18px 16px; line-height: 1.55; box-shadow: 0 10px 28px rgba(0,0,0,.35); }}
.title {{ font-size: 17px; font-weight: 700; margin-bottom: 10px; color: #24b1ff; }}
.dim {{ color: #9fb6cb; font-size: 13px; }}
.code {{ margin-top: 10px; padding: 8px 10px; background: rgba(255,255,255,.04); border-radius: 8px; font-size: 12px; color: #d7e8f8; }}
</style>
</head>
<body>
  <div class=\"card\">
    <div class=\"title\">Neshan map is wired, but the API key is missing</div>
    <div>This shell is ready for the <b>ROAD</b> mode through Neshan.
    The official Neshan web SDK needs a valid API key before the map can open.</div>
    <div class=\"dim\">Put your key in the Python constant <b>NESHAN_WEB_API_KEY</b> or the environment variable of the same name.</div>
    <div class=\"code\">Current request: {style_label}<br>Center: {float(lat):.6f}, {float(lon):.6f}<br>Zoom: {float(zoom):.2f}</div>
  </div>
</body>
</html>"""

    if style_key == 'satellite' and not runtime['satellite_tile_url_present']:
        return f"""<!doctype html>
<html>
<head>
<meta charset=\"utf-8\">
<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no\">
<style>
html, body {{ margin: 0; padding: 0; width: 100%; height: 100%; background: #0d1117; color: #eef6ff; font-family: sans-serif; }}
body {{ display: flex; align-items: center; justify-content: center; }}
.card {{ max-width: 92%; border: 1px solid rgba(56,154,255,.45); background: rgba(17,23,31,.96); border-radius: 14px; padding: 18px 16px; line-height: 1.55; box-shadow: 0 10px 28px rgba(0,0,0,.35); }}
.title {{ font-size: 17px; font-weight: 700; margin-bottom: 10px; color: #24b1ff; }}
.dim {{ color: #9fb6cb; font-size: 13px; }}
.code {{ margin-top: 10px; padding: 8px 10px; background: rgba(255,255,255,.04); border-radius: 8px; font-size: 12px; color: #d7e8f8; }}
</style>
</head>
<body>
  <div class=\"card\">
    <div class=\"title\">Satellite mode placeholder is ready</div>
    <div>The Neshan-first path is active, but Neshan's documented web styles are road/night.
    For <b>SATELLITE</b>, add an Iranian or approved satellite tile URL in <b>IRAN_SATELLITE_TILE_URL</b>.</div>
    <div class=\"dim\">Road mode can use Neshan directly once the API key is set. Satellite waits for the tile source URL.</div>
    <div class=\"code\">Current center: {float(lat):.6f}, {float(lon):.6f}<br>Zoom: {float(zoom):.2f}</div>
  </div>
</body>
</html>"""

    html = f"""<!doctype html>
<html lang=\"fa\">
<head>
<meta charset=\"utf-8\">
<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0, maximum-scale=5.0, user-scalable=yes\">
<link rel=\"stylesheet\" href=\"https://static.neshan.org/sdk/openlayers/v8.1.0/neshan-sdk/v1.0.5/index.css\" />
<style>
html, body, #map {{ margin: 0; padding: 0; width: 100%; height: 100%; background: #101722; overflow: hidden; }}
#crosshair {{ position: fixed; left: 50%; top: 50%; width: 28px; height: 28px; margin-left: -14px; margin-top: -14px; pointer-events: none; z-index: 900; }}
#crosshair:before, #crosshair:after {{ content: ''; position: absolute; background: rgba(36,177,255,.96); }}
#crosshair:before {{ left: 13px; top: 0; width: 2px; height: 28px; }}
#crosshair:after {{ left: 0; top: 13px; width: 28px; height: 2px; }}
#cross-ring {{ position: absolute; left: 4px; top: 4px; width: 18px; height: 18px; border: 2px solid rgba(36,177,255,.96); border-radius: 999px; box-shadow: 0 0 0 1px rgba(0,0,0,.45); }}
.box {{ position: fixed; z-index: 950; color: #eef6ff; background: rgba(11,18,27,.84); border: 1px solid rgba(56,154,255,.35); border-radius: 11px; padding: 7px 10px; font-family: sans-serif; line-height: 1.3; box-shadow: 0 6px 18px rgba(0,0,0,.28); }}
#styleBox {{ left: 10px; top: 10px; font-size: 12px; font-weight: 700; }}
#coordBox {{ left: 10px; bottom: 10px; right: 10px; font-size: 11px; }}
#hintBox {{ right: 10px; top: 10px; font-size: 11px; text-align: right; }}
</style>
<script src=\"https://static.neshan.org/sdk/openlayers/v8.1.0/neshan-sdk/v1.0.5/index.js\"></script>
</head>
<body>
<div id=\"map\"></div>
<div id=\"crosshair\"><div id=\"cross-ring\"></div></div>
<div id=\"styleBox\" class=\"box\">{style_label}</div>
<div id=\"hintBox\" class=\"box\">1 finger: pan<br>2 fingers: pinch zoom</div>
<div id=\"coordBox\" class=\"box\">loading...</div>
<script>
const API_KEY = {api_key_json};
const CUSTOM_SATELLITE_URL = {sat_tile_json};
const START_LAT = {float(lat):.8f};
const START_LON = {float(lon):.8f};
const START_ZOOM = {float(zoom):.2f};
let currentStyleKey = {json.dumps(style_key)};
let currentStyleLabel = {json.dumps(style_label)};
const map = new ol.Map({{
  maptype: 'dreamy',
  target: 'map',
  key: API_KEY,
  view: new ol.View({{
    center: ol.proj.fromLonLat([START_LON, START_LAT]),
    zoom: START_ZOOM
  }})
}});
function utmZone(lat, lon) {{
  let zone = Math.floor((lon + 180) / 6) + 1;
  if (lat >= 56 && lat < 64 && lon >= 3 && lon < 12) zone = 32;
  if (lat >= 72 && lat < 84) {{
    if (lon >= 0 && lon < 9) zone = 31;
    else if (lon < 21) zone = 33;
    else if (lon < 33) zone = 35;
    else if (lon < 42) zone = 37;
  }}
  return Math.max(1, Math.min(60, zone));
}}
function utmBand(lat) {{
  if (lat < -80 || lat > 84) return 'Z';
  const bands = 'CDEFGHJKLMNPQRSTUVWX';
  let idx = Math.floor((lat + 80) / 8);
  idx = Math.max(0, Math.min(idx, bands.length - 1));
  return bands[idx];
}}
function latLonToUtm(lat, lon) {{
  const a = 6378137.0;
  const f = 1 / 298.257223563;
  const e2 = f * (2 - f);
  const ep2 = e2 / (1 - e2);
  const k0 = 0.9996;
  const zone = utmZone(lat, lon);
  const band = utmBand(lat);
  const hemisphere = lat >= 0 ? 'N' : 'S';
  const latRad = lat * Math.PI / 180.0;
  const lonRad = lon * Math.PI / 180.0;
  const lon0 = ((zone - 1) * 6 - 180 + 3) * Math.PI / 180.0;
  const sinLat = Math.sin(latRad);
  const cosLat = Math.cos(latRad);
  const tanLat = Math.tan(latRad);
  const n = a / Math.sqrt(1 - e2 * sinLat * sinLat);
  const t = tanLat * tanLat;
  const c = ep2 * cosLat * cosLat;
  const A = cosLat * (lonRad - lon0);
  const M = a * ((1 - e2 / 4 - 3 * e2 * e2 / 64 - 5 * Math.pow(e2, 3) / 256) * latRad - (3 * e2 / 8 + 3 * e2 * e2 / 32 + 45 * Math.pow(e2, 3) / 1024) * Math.sin(2 * latRad) + (15 * e2 * e2 / 256 + 45 * Math.pow(e2, 3) / 1024) * Math.sin(4 * latRad) - (35 * Math.pow(e2, 3) / 3072) * Math.sin(6 * latRad));
  let easting = k0 * n * (A + (1 - t + c) * Math.pow(A, 3) / 6 + (5 - 18 * t + t * t + 72 * c - 58 * ep2) * Math.pow(A, 5) / 120) + 500000.0;
  let northing = k0 * (M + n * tanLat * (A * A / 2 + (5 - t + 9 * c + 4 * c * c) * Math.pow(A, 4) / 24 + (61 - 58 * t + t * t + 600 * c - 330 * ep2) * Math.pow(A, 6) / 720));
  if (lat < 0) northing += 10000000.0;
  return {{zone, band, hemisphere, easting, northing}};
}}
function applyStyle(styleKey) {{
  currentStyleKey = styleKey === 'satellite' ? 'satellite' : 'road';
  if (currentStyleKey === 'satellite' && CUSTOM_SATELLITE_URL) {{
    map.setMapType(CUSTOM_SATELLITE_URL);
    currentStyleLabel = 'SATELLITE | CUSTOM TILE';
  }} else {{
    map.setMapType('dreamy');
    currentStyleLabel = 'ROAD | NESHAN';
  }}
  document.getElementById('styleBox').textContent = currentStyleLabel;
  updateInfo();
}}
function updateInfo() {{
  const center = ol.proj.toLonLat(map.getView().getCenter());
  const lon = center[0];
  const lat = center[1];
  const zoom = map.getView().getZoom() || START_ZOOM;
  const utm = latLonToUtm(lat, lon);
  document.getElementById('coordBox').innerHTML =
    'LAT: ' + lat.toFixed(6) + ' &nbsp; LON: ' + lon.toFixed(6) + '<br>' +
    'UTM: Z' + utm.zone + utm.band + utm.hemisphere + '  E=' + utm.easting.toFixed(2) + '  N=' + utm.northing.toFixed(2) + '  ZOOM=' + zoom.toFixed(2);
}}
applyStyle(currentStyleKey);
updateInfo();
map.on('moveend', updateInfo);
map.getView().on('change:resolution', updateInfo);
window.setMyLocation = function(lat, lon, zoom) {{
  map.getView().setCenter(ol.proj.fromLonLat([lon, lat]));
  if (zoom !== undefined && zoom !== null) map.getView().setZoom(zoom);
  updateInfo();
}};
window.setMapZoom = function(zoom) {{
  map.getView().setZoom(zoom);
  updateInfo();
}};
window.setMapStyle = function(styleKey) {{
  applyStyle(styleKey);
}};
</script>
</body>
</html>"""
    return html



def build_in_app_map_payload(lat, lon, zoom, style_key='road'):
    lat = float(lat)
    lon = float(lon)
    zoom = max(3.0, min(21.0, float(zoom)))
    style_key = 'satellite' if style_key == 'satellite' else 'road'
    runtime = map_provider_runtime(style_key)

    if style_key == 'satellite':
        if runtime['satellite_tile_url_present']:
            return {
                'mode': 'html',
                'value': build_leaflet_custom_satellite_html(lat, lon, zoom, IRAN_SATELLITE_TILE_URL, 'SATELLITE | IRAN CUSTOM TILE'),
                'label': 'SATELLITE | IRAN CUSTOM TILE',
                'provider': 'iran_custom_satellite',
            }
        return {
            'mode': 'html',
            'value': build_leaflet_map_html(lat, lon, zoom, 'satellite'),
            'label': 'SATELLITE | PUBLIC FALLBACK',
            'provider': 'leaflet_public_satellite',
        }

    if runtime['road_ready']:
        return {
            'mode': 'html',
            'value': build_provider_map_html(lat, lon, zoom, 'road'),
            'label': 'ROAD | NESHAN',
            'provider': 'neshan_road',
        }

    return {
        'mode': 'html',
        'value': build_leaflet_map_html(lat, lon, zoom, 'road'),
        'label': 'ROAD | PUBLIC FALLBACK',
        'provider': 'leaflet_public_road',
    }


def current_runtime_profile():
    executable = str(getattr(sys, 'executable', '') or '')
    argv0 = str(sys.argv[0] if sys.argv else '')
    executable_l = executable.lower()
    argv0_l = argv0.lower()
    package_name = ''
    activity_name = ''
    has_android_activity = False
    try:
        activity, _context, activity_name = get_android_activity_context()
        if activity is not None:
            has_android_activity = True
            try:
                package_name = str(activity.getPackageName() or '')
            except Exception:
                package_name = ''
    except Exception:
        activity_name = ''
        package_name = ''
        has_android_activity = False

    package_name_l = package_name.lower()
    activity_name_l = activity_name.lower()
    pydroid_markers = [
        ('executable', 'pydroid' in executable_l),
        ('argv0', 'pydroid' in argv0_l),
        ('package', 'pydroid' in package_name_l),
        ('activity', 'pydroid' in activity_name_l),
    ]
    reasons = [name for name, ok in pydroid_markers if ok]
    return {
        'is_pydroid_like': bool(reasons),
        'mode_reason': ','.join(reasons) if reasons else ('android_activity' if has_android_activity else 'non_android_runtime'),
        'executable': executable,
        'argv0': argv0,
        'package_name': package_name,
        'activity_name': activity_name,
        'has_android_activity': has_android_activity,
    }


def runtime_profile_probe_text(profile=None):
    profile = profile or current_runtime_profile()
    package_name = profile.get('package_name', '') or '-'
    activity_name = profile.get('activity_name', '') or '-'
    executable = profile.get('executable', '') or '-'
    argv0 = profile.get('argv0', '') or '-'
    return (
        f"runtime_mode={'pydroid_safe' if profile.get('is_pydroid_like') else ('apk_target' if profile.get('has_android_activity') else 'non_android_safe')}\n"
        f"reason={profile.get('mode_reason', '-') or '-'}\n"
        f"package={package_name}\n"
        f"activity={activity_name}\n"
        f"executable={executable}\n"
        f"argv0={argv0}"
    )


MAP_RUNTIME_TARGETS = {
    'road': {
        'provider': 'NESHAN',
        'title': 'APK TARGET | ROAD SECONDARY',
        'body': 'Satellite-first branch is active. ROAD stays secondary until in-app satellite is visible in the packaged APK runtime.',
        'style_label': 'NESHAN ROAD',
        'ui_style_key': 'road',
    },
    'satellite': {
        'provider': 'SATELLITE',
        'title': 'APK TARGET | SATELLITE FIRST',
        'body': 'Primary target: bring the in-app satellite map up first in the packaged APK runtime. ROAD can follow after that.',
        'style_label': 'SATELLITE TARGET',
        'ui_style_key': 'satellite',
    },
}


def map_runtime_target(style_key='road'):
    style_key = 'satellite' if style_key == 'satellite' else 'road'
    data = dict(MAP_RUNTIME_TARGETS[style_key])
    data['style_key'] = style_key
    return data


def runtime_map_mode():
    profile = current_runtime_profile()
    if profile.get('has_android_activity'):
        return 'android_inapp'
    return 'non_android_safe'


def runtime_map_message(target):
    mode = runtime_map_mode()
    profile = current_runtime_profile()
    if mode == 'non_android_safe':
        return (
            target['title'],
            'RUNTIME: NON-ANDROID SAFE',
            'SAFE MODE: Android map runtime not available here',
            'This runtime is not an Android activity, so the in-app map cannot be attempted here.',
            f"GPS > {target['style_key'].upper()} | non-android safe mode",
        )
    if profile.get('is_pydroid_like'):
        if target['style_key'] == 'satellite':
            return (
                target['title'],
                'RUNTIME: PYDROID ANDROID IN-APP TRY',
                'SAT will now attempt to open inside the app again',
                'This branch no longer blocks SAT/ROAD in Pydroid-like Android runtime. If WebView creation still fails, use the exact on-screen error text or inapp_webview_error.txt.',
                'GPS > SATELLITE | Pydroid in-app map attempt',
            )
        return (
            target['title'],
            'RUNTIME: PYDROID ANDROID IN-APP TRY',
            'ROAD will now attempt to open inside the app again',
            'This branch no longer blocks SAT/ROAD in Pydroid-like Android runtime. If WebView creation still fails, use the exact on-screen error text or inapp_webview_error.txt.',
            'GPS > ROAD | Pydroid in-app map attempt',
        )
    if target['style_key'] == 'satellite':
        return (
            target['title'],
            'RUNTIME: ANDROID SATELLITE-IN-APP PATH',
            'ANDROID BUILD SHOULD OPEN IN-APP SATELLITE HERE',
            'Primary success criterion remains: SATELLITE must appear inside the app frame.',
            'GPS > SATELLITE | Android in-app map path ready',
        )
    return (
        target['title'],
        'RUNTIME: ANDROID ROAD-IN-APP PATH',
        'ANDROID BUILD SHOULD OPEN IN-APP ROAD HERE',
        'ROAD remains secondary. It matters only after SATELLITE appears inside the app frame.',
        'GPS > ROAD | Android in-app map path ready',
    )


class _AndroidCloseClick(PythonJavaClass):
    __javainterfaces__ = ['android/view/View$OnClickListener']

    def __init__(self, callback):
        super().__init__()
        self._callback = callback

    @java_method('(Landroid/view/View;)V')
    def onClick(self, _view):
        if self._callback:
            self._callback()


class NullMapRuntime:
    def __init__(self):
        self.is_visible = False
        self.current_style_key = 'road'

    def show(self, *args, **kwargs):
        self.is_visible = False
        return False

    def set_style(self, *args, **kwargs):
        return False

    def set_zoom(self, *args, **kwargs):
        return False

    def update_viewport(self, *args, **kwargs):
        return False

    def hide(self):
        self.is_visible = False
        return False


class AndroidWebMapOverlay:
    """Packaged-APK native WebView overlay for the in-app map area only."""
    def __init__(self, status_callback=None):
        self.status_callback = status_callback
        self.available = False
        self.activity = None
        self.webview = None
        self.container = None
        self.root = None
        self.is_visible = False
        self.current_style_key = 'road'
        self.current_lat = None
        self.current_lon = None
        self.current_zoom = GPS_PARCEL_ZOOM if 'GPS_PARCEL_ZOOM' in globals() else TILE_ZOOM
        self.viewport = None
        self.last_report = 'APK native map pending'
        self.last_payload = None
        self._webview_create_report = 'not_tried'
        self._detect_runtime()

    def _report(self, text):
        self.last_report = str(text)
        if self.status_callback:
            try:
                self.status_callback(text)
            except Exception:
                pass

    def _write_diagnostic_file(self, title, detail):
        try:
            diagnostic_path = os.path.join(BASE_DIR, 'inapp_webview_error.txt')
            with open(diagnostic_path, 'w', encoding='utf-8') as fh:
                fh.write(title + '\n\n' + detail)
        except Exception:
            pass

    def _fail(self, step, exc):
        detail = (
            f'step={step}\n'
            f'exception={exc.__class__.__name__}: {exc}\n'
            f'webview_create_report={self._webview_create_report}\n'
            f'style={self.current_style_key}\n'
            f'lat={self.current_lat}\n'
            f'lon={self.current_lon}\n'
            f'zoom={self.current_zoom}\n\n'
            f'{traceback.format_exc()}'
        )
        self._write_diagnostic_file('IN-APP WEBVIEW ERROR', detail)
        self._clear_runtime_widgets()
        self.is_visible = False
        self._report(f'In-app map error | {step} | {exc.__class__.__name__}: {exc}')

    def _detect_runtime(self):
        try:
            activity, _context, _activity_name = get_android_activity_context()
            self.activity = activity
            self.available = activity is not None
        except Exception:
            self.available = False
            self.activity = None

    def _clear_runtime_widgets(self):
        self.webview = None
        self.container = None
        self.root = None

    def show(self, lat, lon, zoom=None, style_key=None, viewport=None):
        self.current_lat = float(lat)
        self.current_lon = float(lon)
        if zoom is not None:
            self.current_zoom = float(zoom)
        if style_key is not None:
            self.current_style_key = 'satellite' if style_key == 'satellite' else 'road'
        if viewport is not None:
            self.viewport = dict(viewport)
        self._detect_runtime()
        if not self.available:
            self.is_visible = False
            self._report('Android activity/WebView runtime is not available here')
            return False
        self._show_on_ui_thread()
        return True

    def set_style(self, style_key):
        self.current_style_key = 'satellite' if style_key == 'satellite' else 'road'
        if not self.available or self.webview is None or self.current_lat is None or self.current_lon is None:
            return False
        self._reload_on_ui_thread()
        return True

    def set_zoom(self, zoom):
        try:
            self.current_zoom = float(zoom)
        except Exception:
            return False
        if not self.available or self.webview is None or self.current_lat is None or self.current_lon is None:
            return False
        self._reload_on_ui_thread()
        return True

    def update_viewport(self, viewport):
        if viewport is not None:
            self.viewport = dict(viewport)
        if not self.available or self.container is None:
            return False
        self._apply_viewport_on_ui_thread()
        return True

    def hide(self):
        if self.container is None:
            self.is_visible = False
            return False
        self._hide_on_ui_thread()
        return True

    def dispose(self):
        if self.container is None and self.webview is None:
            self.is_visible = False
            return False
        self._dispose_on_ui_thread()
        return True

    def _context_candidates(self, activity):
        candidates = []
        seen = set()

        def add(label, ctx):
            if ctx is None:
                return
            marker = id(ctx)
            if marker in seen:
                return
            seen.add(marker)
            candidates.append((label, ctx))

        add('activity', activity)
        try:
            add('application_context', activity.getApplicationContext())
        except Exception:
            pass
        try:
            window = activity.getWindow()
            decor = window.getDecorView() if window is not None else None
            add('decor_context', decor.getContext() if decor is not None else None)
        except Exception:
            pass
        try:
            root = activity.findViewById(16908290)
            add('content_root_context', root.getContext() if root is not None else None)
        except Exception:
            pass
        return candidates

    def _create_webview_with_fallbacks(self, autoclass, WebView, activity):
        errors = []
        for label, ctx in self._context_candidates(activity):
            try:
                webview = WebView(ctx)
                self._webview_create_report = f'constructor={label}'
                return webview
            except Exception as exc:
                errors.append(f'{label}: {exc.__class__.__name__}: {exc}')

        try:
            ContextThemeWrapper = autoclass('android.view.ContextThemeWrapper')
            AndroidRStyle = autoclass('android.R$style')
            themed_ctx = ContextThemeWrapper(activity, AndroidRStyle.Theme_DeviceDefault)
            webview = WebView(themed_ctx)
            self._webview_create_report = 'constructor=themed_device_default'
            return webview
        except Exception as exc:
            errors.append(f'themed_device_default: {exc.__class__.__name__}: {exc}')

        self._webview_create_report = ' | '.join(errors) if errors else 'no_constructor_attempt_logged'
        raise RuntimeError('WebView constructor failed across fallback contexts')

    def _coerce_viewport(self, activity):
        display = activity.getResources().getDisplayMetrics()
        density = float(display.density)
        screen_w = int(display.widthPixels)
        screen_h = int(display.heightPixels)
        default = {
            'left': int(92 * density),
            'top': int(64 * density),
            'right': int(8 * density),
            'bottom': int(38 * density),
        }
        if not isinstance(self.viewport, dict):
            return default, screen_w, screen_h
        vp = dict(default)
        for key in ('left', 'top', 'right', 'bottom'):
            try:
                vp[key] = int(float(self.viewport.get(key, vp[key])))
            except Exception:
                pass
        vp['left'] = max(0, min(screen_w - int(80 * density), vp['left']))
        vp['top'] = max(0, min(screen_h - int(80 * density), vp['top']))
        vp['right'] = max(0, min(screen_w - vp['left'] - int(40 * density), vp['right']))
        vp['bottom'] = max(0, min(screen_h - vp['top'] - int(40 * density), vp['bottom']))
        return vp, screen_w, screen_h

    @run_on_ui_thread
    def _apply_viewport_on_ui_thread(self):
        try:
            from jnius import autoclass
            activity, _context, _activity_name = get_android_activity_context()
            if activity is None or self.container is None:
                return
            self.activity = activity
            FrameLayoutParams = autoclass('android.widget.FrameLayout$LayoutParams')
            Gravity = autoclass('android.view.Gravity')
            vp, screen_w, screen_h = self._coerce_viewport(activity)
            width = max(160, screen_w - vp['left'] - vp['right'])
            height = max(160, screen_h - vp['top'] - vp['bottom'])
            params = FrameLayoutParams(width, height)
            params.gravity = Gravity.TOP | Gravity.LEFT
            params.setMargins(vp['left'], vp['top'], vp['right'], vp['bottom'])
            self.container.setLayoutParams(params)
            self.container.requestLayout()
            try:
                self.container.bringToFront()
            except Exception:
                pass
        except Exception as exc:
            self._report(f'In-app map viewport error: {exc.__class__.__name__}: {exc}')

    @run_on_ui_thread
    def _show_on_ui_thread(self):
        step = 'begin'
        try:
            from jnius import autoclass
            step = 'get activity'
            activity, _context, _activity_name = get_android_activity_context()
            if activity is None:
                raise RuntimeError('Android activity is not available for in-app map')
            self.activity = activity

            step = 'resolve Android classes'
            WebView = autoclass('android.webkit.WebView')
            WebViewClient = autoclass('android.webkit.WebViewClient')
            FrameLayout = autoclass('android.widget.FrameLayout')
            FrameLayoutParams = autoclass('android.widget.FrameLayout$LayoutParams')
            ViewGroupLayoutParams = autoclass('android.view.ViewGroup$LayoutParams')
            Gravity = autoclass('android.view.Gravity')
            View = autoclass('android.view.View')
            ColorClass = autoclass('android.graphics.Color')

            if self.container is None or self.webview is None:
                step = 'create container'
                container = FrameLayout(activity)
                try:
                    container.setBackgroundColor(ColorClass.parseColor('#101722'))
                except Exception:
                    pass

                step = 'create webview'
                webview = self._create_webview_with_fallbacks(autoclass, WebView, activity)
                step = 'read webview settings'
                settings = webview.getSettings()
                if settings is None:
                    raise RuntimeError('WebView settings unavailable')

                step = 'configure webview settings'
                settings.setJavaScriptEnabled(True)
                settings.setDomStorageEnabled(True)
                settings.setSupportZoom(True)
                try:
                    settings.setBuiltInZoomControls(False)
                    settings.setDisplayZoomControls(False)
                except Exception:
                    pass
                try:
                    settings.setUseWideViewPort(True)
                    settings.setLoadWithOverviewMode(True)
                except Exception:
                    pass
                try:
                    settings.setAllowFileAccess(True)
                    settings.setAllowContentAccess(True)
                    settings.setDatabaseEnabled(True)
                except Exception:
                    pass
                try:
                    settings.setLoadsImagesAutomatically(True)
                except Exception:
                    pass
                try:
                    settings.setJavaScriptCanOpenWindowsAutomatically(False)
                except Exception:
                    pass
                try:
                    WebSettings = autoclass('android.webkit.WebSettings')
                    settings.setMixedContentMode(WebSettings.MIXED_CONTENT_COMPATIBILITY_MODE)
                except Exception:
                    pass
                try:
                    webview.setBackgroundColor(ColorClass.parseColor('#101722'))
                except Exception:
                    pass
                try:
                    webview.setVerticalScrollBarEnabled(False)
                    webview.setHorizontalScrollBarEnabled(False)
                except Exception:
                    pass

                step = 'assign webview client'
                webview.setWebViewClient(WebViewClient())
                step = 'attach webview to container'
                container.addView(webview, FrameLayoutParams(ViewGroupLayoutParams.MATCH_PARENT, ViewGroupLayoutParams.MATCH_PARENT))
                self.container = container
                self.webview = webview

            step = 'build payload'
            payload = build_in_app_map_payload(self.current_lat, self.current_lon, self.current_zoom, self.current_style_key)
            self.last_payload = dict(payload)

            step = 'compute viewport'
            vp, screen_w, screen_h = self._coerce_viewport(activity)
            width = max(160, screen_w - vp['left'] - vp['right'])
            height = max(160, screen_h - vp['top'] - vp['bottom'])
            params = FrameLayoutParams(width, height)
            params.gravity = Gravity.TOP | Gravity.LEFT
            params.setMargins(vp['left'], vp['top'], vp['right'], vp['bottom'])

            step = 'find android content root'
            root = activity.findViewById(16908290)
            if root is None:
                raise RuntimeError('Android content root was not found')
            self.root = root

            step = 'attach container to root'
            if self.container.getParent() is None:
                root.addView(self.container, params)
            else:
                self.container.setLayoutParams(params)

            step = 'bring container to front'
            try:
                self.container.bringToFront()
            except Exception:
                pass

            step = 'load payload into webview'
            try:
                self.webview.clearCache(True)
            except Exception:
                pass
            if payload['mode'] == 'url':
                self.webview.loadUrl(payload['value'])
            else:
                self.webview.loadDataWithBaseURL('https://appassets.androidplatform.net/assets/', payload['value'], 'text/html', 'utf-8', None)

            step = 'show container'
            self.container.setVisibility(View.VISIBLE)
            self.is_visible = True
            self._report(f"In-app map opened | {payload['label']}")
        except Exception as exc:
            self._fail(step, exc)

    @run_on_ui_thread
    def _reload_on_ui_thread(self):
        step = 'reload begin'
        if self.webview is None or self.current_lat is None or self.current_lon is None:
            return
        try:
            step = 'build payload'
            payload = build_in_app_map_payload(self.current_lat, self.current_lon, self.current_zoom, self.current_style_key)
            self.last_payload = dict(payload)
            step = 'reload payload into webview'
            try:
                self.webview.clearCache(True)
            except Exception:
                pass
            if payload['mode'] == 'url':
                self.webview.loadUrl(payload['value'])
            else:
                self.webview.loadDataWithBaseURL('https://appassets.androidplatform.net/assets/', payload['value'], 'text/html', 'utf-8', None)
            self.is_visible = True
            self._report(f"In-app map reloaded | {payload['label']}")
        except Exception as exc:
            self._fail(step, exc)

    @run_on_ui_thread
    def _hide_on_ui_thread(self):
        try:
            from jnius import autoclass
            View = autoclass('android.view.View')
            if self.webview is not None:
                try:
                    self.webview.stopLoading()
                except Exception:
                    pass
            if self.container is not None:
                self.container.setVisibility(View.GONE)
            self.is_visible = False
            self._report('In-app map hidden')
        except Exception as exc:
            self._report(f'In-app map close error: {exc.__class__.__name__}: {exc}')

    @run_on_ui_thread
    def _dispose_on_ui_thread(self):
        try:
            from jnius import autoclass
            View = autoclass('android.view.View')
            if self.webview is not None:
                try:
                    self.webview.stopLoading()
                except Exception:
                    pass
                try:
                    self.webview.loadUrl('about:blank')
                except Exception:
                    pass
                try:
                    self.webview.clearHistory()
                except Exception:
                    pass
                try:
                    self.webview.removeAllViews()
                except Exception:
                    pass
                try:
                    self.webview.destroy()
                except Exception:
                    pass
            if self.container is not None:
                try:
                    parent = self.container.getParent()
                    if parent is not None:
                        parent.removeView(self.container)
                except Exception:
                    pass
                try:
                    self.container.setVisibility(View.GONE)
                    self.container.removeAllViews()
                except Exception:
                    pass
            self._clear_runtime_widgets()
            self.is_visible = False
        except Exception as exc:
            self._report(f'In-app map dispose error: {exc.__class__.__name__}: {exc}')


def module_available(name):
    try:
        return importlib.util.find_spec(name) is not None
    except Exception:
        return False


def get_android_bridge():
    for module_name in ('androidhelper', 'android'):
        try:
            module = __import__(module_name)
            bridge = getattr(module, 'Android')()
            return bridge, module_name
        except Exception:
            pass
    return None, None


def get_android_activity_context():
    try:
        from jnius import autoclass, cast
    except Exception:
        return None, None, None

    for activity_class_name in ('org.kivy.android.PythonActivity', 'org.renpy.android.PythonActivity'):
        try:
            PythonActivity = autoclass(activity_class_name)
            activity = cast('android.app.Activity', PythonActivity.mActivity)
            context = cast('android.content.Context', activity.getApplicationContext())
            return activity, context, activity_class_name
        except Exception:
            pass
    return None, None, None




def build_external_map_targets(lat, lon, zoom=19, style_key='road'):
    lat = float(lat)
    lon = float(lon)
    zoom = max(3, min(21, float(zoom)))
    lat_txt = f'{lat:.6f}'
    lon_txt = f'{lon:.6f}'
    zoom_txt = f'{zoom:.1f}'
    if style_key == 'road':
        return [
            {
                'label': 'Neshan short link',
                'uri': f'https://nshn.ir/?lat={lat_txt}&lng={lon_txt}',
            },
            {
                'label': 'Neshan web map',
                'uri': f'https://maps.neshan.org/@{lat_txt},{lon_txt},{zoom_txt}z,0.0p',
            },
        ]
    return [
        {
            'label': 'Google Maps satellite URL',
            'uri': f'https://www.google.com/maps/@?api=1&map_action=map&center={lat_txt},{lon_txt}&zoom={int(round(zoom))}&basemap=satellite',
        },
        {
            'label': 'Google Maps coordinate URL',
            'uri': f'https://www.google.com/maps/@{lat_txt},{lon_txt},{zoom_txt}z/data=!3m1!1e3',
        },
    ]


def try_open_external_map(style_key, lat, lon, zoom=19):
    targets = build_external_map_targets(lat, lon, zoom=zoom, style_key=style_key)
    errors = []

    activity, _context, _activity_name = get_android_activity_context()
    if activity is not None:
        try:
            from jnius import autoclass
            Intent = autoclass('android.content.Intent')
            Uri = autoclass('android.net.Uri')
            for target in targets:
                intent = Intent(Intent.ACTION_VIEW, Uri.parse(target['uri']))
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                activity.startActivity(intent)
                return {
                    'ok': True,
                    'mode': 'android_intent',
                    'label': target['label'],
                    'uri': target['uri'],
                    'message': f"Opened external map via {target['label']}",
                }
        except Exception as exc:
            errors.append(f'android_intent: {exc.__class__.__name__}: {exc}')

    for target in targets:
        try:
            opened = webbrowser.open(target['uri'])
            if opened:
                return {
                    'ok': True,
                    'mode': 'webbrowser',
                    'label': target['label'],
                    'uri': target['uri'],
                    'message': f"Opened external map via {target['label']}",
                }
        except Exception as exc:
            errors.append(f"{target['label']}: {exc.__class__.__name__}: {exc}")

    detail = ' | '.join(errors) if errors else 'no launcher succeeded'
    return {
        'ok': False,
        'mode': 'none',
        'label': '',
        'uri': targets[0]['uri'] if targets else '',
        'message': f'External map launch failed: {detail}',
        'detail': detail,
    }


def request_device_location_permissions():
    requested = False
    try:
        from android.permissions import request_permissions, Permission
        request_permissions([Permission.ACCESS_FINE_LOCATION, Permission.ACCESS_COARSE_LOCATION])
        requested = True
    except Exception:
        pass

    bridge, _module_name = get_android_bridge()
    if bridge is not None:
        for method_name in ('requestPermissions', 'requestpermissions'):
            method = getattr(bridge, method_name, None)
            if method is None:
                continue
            try:
                method([
                    'android.permission.ACCESS_FINE_LOCATION',
                    'android.permission.ACCESS_COARSE_LOCATION',
                ])
                requested = True
                break
            except Exception:
                pass
    return requested


def normalize_location_source_name(source_name):
    text = str(source_name or 'PYJNIUS').upper()
    if text.endswith('_PROVIDER'):
        text = text[:-9]
    if '.' in text:
        text = text.split('.')[-1]
    if text == 'FUSEDLOCATIONPROVIDERCLIENT':
        text = 'FUSED'
    return text


def location_fix_rank(item):
    priority = {'GPS': 0, 'FUSED': 1, 'NETWORK': 2, 'PASSIVE': 3, 'DIRECT': 4, 'PLYER': 5, 'IP': 6}
    acc = item[2]
    if acc <= 0:
        acc = 999999.0
    return (priority.get(item[3], 9), acc)


def better_location_fix(current, candidate):
    if candidate is None:
        return current
    if current is None:
        return candidate
    return candidate if location_fix_rank(candidate) < location_fix_rank(current) else current


def is_good_device_fix(fix):
    if fix is None:
        return False
    source = fix[3]
    acc = fix[2]
    if source == 'GPS':
        return True
    return 0 < acc <= 30


def fix_from_java_location(location, provider_hint=None):
    if location is None:
        return None
    try:
        lat = float(location.getLatitude())
        lon = float(location.getLongitude())
    except Exception:
        return None
    try:
        has_acc = location.hasAccuracy()
    except Exception:
        has_acc = False
    try:
        acc = float(location.getAccuracy()) if has_acc else 0.0
    except Exception:
        acc = 0.0
    provider_name = provider_hint
    if provider_name is None:
        try:
            provider_name = location.getProvider()
        except Exception:
            provider_name = 'pyjnius'
    return (lat, lon, acc, normalize_location_source_name(provider_name))


def read_last_known_locations_pyjnius(manager, providers):
    best_fix = None
    for provider in providers:
        try:
            location = manager.getLastKnownLocation(provider)
            fix = fix_from_java_location(location, provider)
            best_fix = better_location_fix(best_fix, fix)
        except Exception:
            pass
    return best_fix


def try_get_pyjnius_last_known_fix():
    try:
        from jnius import autoclass, cast
    except Exception as exc:
        return None, f'pyjnius missing: {exc.__class__.__name__}'

    activity, _context, activity_name = get_android_activity_context()
    if activity is None:
        return None, 'pyjnius activity unavailable'

    try:
        Context = autoclass('android.content.Context')
        LocationManager = autoclass('android.location.LocationManager')
        manager = cast('android.location.LocationManager', activity.getSystemService(Context.LOCATION_SERVICE))
    except Exception as exc:
        return None, f'pyjnius manager failed: {exc.__class__.__name__}'

    provider_candidates = [LocationManager.GPS_PROVIDER, LocationManager.NETWORK_PROVIDER, LocationManager.PASSIVE_PROVIDER]
    active_providers = []
    provider_state = []
    for provider in provider_candidates:
        try:
            enabled = bool(manager.isProviderEnabled(provider))
        except Exception:
            enabled = False
        provider_state.append(f'{provider}={enabled}')
        if enabled:
            active_providers.append(provider)
    best_fix = read_last_known_locations_pyjnius(manager, active_providers or provider_candidates)
    report = f'activity={activity_name} | ' + '; '.join(provider_state)
    return best_fix, report


def begin_pyjnius_live_session(timeout_sec=10):
    try:
        from jnius import autoclass, cast, PythonJavaClass, java_method
    except Exception as exc:
        return None, '', f'pyjnius missing: {exc.__class__.__name__}'

    activity, _context, activity_name = get_android_activity_context()
    if activity is None:
        return None, '', 'pyjnius activity unavailable'

    try:
        Context = autoclass('android.content.Context')
        LocationManager = autoclass('android.location.LocationManager')
        Looper = autoclass('android.os.Looper')
        manager = cast('android.location.LocationManager', activity.getSystemService(Context.LOCATION_SERVICE))
    except Exception as exc:
        return None, '', f'pyjnius manager failed: {exc.__class__.__name__}'

    if manager is None:
        return None, '', 'pyjnius manager is None'

    provider_candidates = [LocationManager.GPS_PROVIDER, LocationManager.NETWORK_PROVIDER, LocationManager.PASSIVE_PROVIDER]
    active_providers = []
    provider_state = []
    for provider in provider_candidates:
        try:
            enabled = bool(manager.isProviderEnabled(provider))
        except Exception:
            enabled = False
        provider_state.append(f'{provider}={enabled}')
        if enabled:
            active_providers.append(provider)
    if not active_providers:
        active_providers = provider_candidates[:]

    best_fix = read_last_known_locations_pyjnius(manager, active_providers)
    holder = {'best': best_fix}

    class JniusLocationListener(PythonJavaClass):
        __javainterfaces__ = ['android/location/LocationListener']
        __javacontext__ = 'app'

        def __init__(self):
            super().__init__()

        @java_method('(Landroid/location/Location;)V')
        def onLocationChanged(self, location):
            fix = fix_from_java_location(location, None)
            holder['best'] = better_location_fix(holder.get('best'), fix)

        @java_method('(Ljava/lang/String;)V')
        def onProviderDisabled(self, provider):
            return None

        @java_method('(Ljava/lang/String;)V')
        def onProviderEnabled(self, provider):
            return None

        @java_method('(Ljava/lang/String;ILandroid/os/Bundle;)V')
        def onStatusChanged(self, provider, status, extras):
            return None

    listener = JniusLocationListener()
    try:
        looper = Looper.getMainLooper()
    except Exception:
        looper = None

    registered = False
    request_errors = []
    for provider in active_providers:
        try:
            if looper is not None:
                manager.requestLocationUpdates(provider, 1000, 0, listener, looper)
            else:
                manager.requestLocationUpdates(provider, 1000, 0, listener)
            registered = True
        except Exception as exc:
            request_errors.append(f'{provider}:{exc.__class__.__name__}')

    report = f'activity={activity_name} | ' + '; '.join(provider_state)
    if request_errors:
        report += ' | request_errors=' + ', '.join(request_errors)

    session = {
        'manager': manager,
        'listener': listener,
        'providers': active_providers,
        'holder': holder,
        'best': best_fix,
        'report': report,
        'deadline': time.time() + max(8, timeout_sec),
        'registered': registered,
    }
    return session, report, None


def poll_pyjnius_live_session(session):
    manager = session.get('manager')
    providers = session.get('providers') or []
    best_fix = session.get('best')
    best_fix = better_location_fix(best_fix, session.get('holder', {}).get('best'))
    if manager is not None and providers:
        refreshed = read_last_known_locations_pyjnius(manager, providers)
        best_fix = better_location_fix(best_fix, refreshed)
    session['best'] = best_fix
    return best_fix


def try_get_pyjnius_device_location(timeout_sec=20):
    session, report, error = begin_pyjnius_live_session(timeout_sec=timeout_sec)
    if session is None:
        return None, report or error or 'pyjnius live session unavailable'

    deadline = time.time() + max(5.0, float(timeout_sec))
    best_fix = session.get('best')
    try:
        while time.time() < deadline:
            best_fix = better_location_fix(best_fix, poll_pyjnius_live_session(session))
            if is_good_device_fix(best_fix):
                break
            time.sleep(0.45)
    finally:
        close_pyjnius_live_session(session)
    return best_fix, report


def close_pyjnius_live_session(session):
    manager = session.get('manager')
    listener = session.get('listener')
    if manager is None or listener is None:
        return
    try:
        manager.removeUpdates(listener)
    except Exception:
        pass


def normalize_location_payload(payload):
    if hasattr(payload, 'result'):
        try:
            payload = payload.result
        except Exception:
            pass
    if isinstance(payload, str):
        text = payload.strip()
        if text:
            for loader in (json.loads, ast.literal_eval):
                try:
                    return normalize_location_payload(loader(text))
                except Exception:
                    pass
    return payload


def parse_location_bundle(bundle):
    bundle = normalize_location_payload(bundle)
    candidates = []

    def add_candidate(source_name, info):
        if not isinstance(info, dict):
            return
        if 'latitude' not in info or 'longitude' not in info:
            return
        try:
            lat = float(info['latitude'])
            lon = float(info['longitude'])
            acc = float(info.get('accuracy', 0.0) or 0.0)
            candidates.append((lat, lon, acc, str(source_name).upper()))
        except Exception:
            pass

    if isinstance(bundle, dict):
        if 'latitude' in bundle and 'longitude' in bundle:
            add_candidate(bundle.get('provider', 'direct'), bundle)
        for key, value in bundle.items():
            add_candidate(key, value)

    if not candidates:
        return None
    return min(candidates, key=location_fix_rank)


def read_android_location_snapshot(bridge):
    best_fix = None
    for method_name in ('readLocation', 'getLastKnownLocation'):
        method = getattr(bridge, method_name, None)
        if method is None:
            continue
        try:
            result = method().result
            fix = parse_location_bundle(result)
            best_fix = better_location_fix(best_fix, fix)
        except Exception:
            pass
    return best_fix


def clear_android_location_events(bridge):
    for args in [(), ('location',)]:
        method = getattr(bridge, 'eventClearBuffer', None)
        if method is None:
            break
        try:
            method(*args)
            return
        except Exception:
            pass


def extract_fix_from_android_event(payload):
    payload = normalize_location_payload(payload)
    if payload is None:
        return None
    if isinstance(payload, list):
        for item in payload:
            fix = extract_fix_from_android_event(item)
            if fix is not None:
                return fix
        return None
    if not isinstance(payload, dict):
        return None
    name = str(payload.get('name', '')).lower()
    data = payload.get('data', payload)
    if 'location' in name:
        fix = parse_location_bundle(data)
        if fix is not None:
            return fix
    return parse_location_bundle(data)


def wait_android_location_event(bridge, timeout_ms=2500):
    for method_name in ('eventWaitFor', 'eventPoll'):
        method = getattr(bridge, method_name, None)
        if method is None:
            continue
        try:
            if method_name == 'eventWaitFor':
                result = method('location', timeout_ms).result
            else:
                result = method(1).result
            time.sleep(0.12)
            fix = extract_fix_from_android_event(result)
            if fix is not None:
                return fix
        except Exception:
            pass
    return None


def start_android_location_updates(bridge):
    attempts = [
        ((), {'minDistance': 1000, 'minUpdateDistance': 1}),
        ((1000, 1), {}),
        ((), {'minDistance': 2000, 'minUpdateDistance': 0}),
        ((2000, 0), {}),
        ((1000, 0), {}),
        ((), {}),
    ]
    for args, kwargs in attempts:
        try:
            bridge.startLocating(*args, **kwargs)
            return True
        except Exception:
            pass
    return False


def stop_android_location_updates(bridge):
    method = getattr(bridge, 'stopLocating', None)
    if method is None:
        return
    try:
        method()
    except Exception:
        pass


def try_get_android_device_location(timeout_sec=12):
    bridge, module_name = get_android_bridge()
    if bridge is None:
        return None, 'android bridge unavailable', module_name

    provider_info = 'providers=unknown'
    try:
        providers = getattr(bridge, 'locationProviders', None)
        gps_enabled = getattr(bridge, 'locationProviderEnabled', None)
        plist = providers().result if providers is not None else None
        gps_ok = gps_enabled('gps').result if gps_enabled is not None else None
        net_ok = gps_enabled('network').result if gps_enabled is not None else None
        provider_info = f'providers={plist} gps={gps_ok} net={net_ok}'
    except Exception:
        pass

    clear_android_location_events(bridge)
    best_fix = read_android_location_snapshot(bridge)
    active_started = start_android_location_updates(bridge)
    deadline = time.time() + max(12, timeout_sec)

    try:
        while time.time() < deadline:
            remaining_ms = max(1200, int((deadline - time.time()) * 1000))
            fix = wait_android_location_event(bridge, timeout_ms=min(5000, remaining_ms))
            best_fix = better_location_fix(best_fix, fix)
            fix = read_android_location_snapshot(bridge)
            best_fix = better_location_fix(best_fix, fix)
            if is_good_device_fix(best_fix):
                return best_fix, provider_info, module_name
            time.sleep(0.2 if active_started else 0.5)
    finally:
        stop_android_location_updates(bridge)

    return best_fix, provider_info, module_name


def probe_runtime_modules():
    bridge, bridge_module = get_android_bridge()
    info = {
        'androidhelper': module_available('androidhelper'),
        'android': module_available('android'),
        'plyer': module_available('plyer'),
        'jnius': module_available('jnius'),
        'bridge': bridge is not None,
        'bridge_module': bridge_module or 'none',
    }
    parts = [
        f"androidhelper={'yes' if info['androidhelper'] else 'no'}",
        f"android={'yes' if info['android'] else 'no'}",
        f"plyer={'yes' if info['plyer'] else 'no'}",
        f"jnius={'yes' if info['jnius'] else 'no'}",
        f"bridge={'yes' if info['bridge'] else 'no'}",
        f"bridge_module={info['bridge_module']}",
    ]
    info['summary'] = ' | '.join(parts)
    info['multiline'] = '\n'.join([
        f"androidhelper: {'yes' if info['androidhelper'] else 'no'}",
        f"android: {'yes' if info['android'] else 'no'}",
        f"plyer: {'yes' if info['plyer'] else 'no'}",
        f"jnius: {'yes' if info['jnius'] else 'no'}",
        f"bridge: {'yes' if info['bridge'] else 'no'}",
        f"bridge_module: {info['bridge_module']}",
    ])
    return info


def fetch_live_location():
    request_device_location_permissions()
    probe = probe_runtime_modules()
    detail_lines = []

    last_known_fix, last_known_report = try_get_pyjnius_last_known_fix()
    if last_known_report:
        detail_lines.append(f'pyjnius_last_known: {last_known_report}')

    live_fix, live_report = try_get_pyjnius_device_location(timeout_sec=20)
    if live_report:
        detail_lines.append(f'pyjnius_live: {live_report}')
    best_fix = better_location_fix(last_known_fix, live_fix)
    if is_good_device_fix(best_fix):
        return {
            'ok': True,
            'lat': best_fix[0],
            'lon': best_fix[1],
            'accuracy': best_fix[2],
            'provider': best_fix[3],
            'backend': 'pyjnius',
            'probe': probe,
            'detail': '\n'.join(detail_lines),
            'message': 'Live GPS fix received via PyJNIus',
        }

    bridge_fix, bridge_report, bridge_module = try_get_android_device_location(timeout_sec=12)
    if bridge_report:
        detail_lines.append(f'android_bridge: {bridge_report}')
    best_fix = better_location_fix(best_fix, bridge_fix)
    if best_fix is not None:
        backend = 'pyjnius' if best_fix in (last_known_fix, live_fix) else (bridge_module or 'android')
        quality = 'good' if is_good_device_fix(best_fix) else 'weak'
        return {
            'ok': True,
            'lat': best_fix[0],
            'lon': best_fix[1],
            'accuracy': best_fix[2],
            'provider': best_fix[3],
            'backend': backend,
            'probe': probe,
            'detail': '\n'.join(detail_lines),
            'message': f'{quality} GPS fix received',
            'weak_fix': not is_good_device_fix(best_fix),
        }

    return {
        'ok': False,
        'message': 'No device location fix available - check GPS permission and phone location settings',
        'probe': probe,
        'detail': '\n'.join(detail_lines),
    }


class FlatMenuButton(Button):
    def __init__(self, text='', active=False, **kwargs):
        kwargs.setdefault('size_hint', (1, None))
        kwargs.setdefault('height', TOP_H - dp(16))
        super().__init__(text=text, **kwargs)
        self.background_normal = ''
        self.background_down = ''
        self.border = (0, 0, 0, 0)
        self.color = FG_MAIN
        self.font_size = '9sp'
        self.bold = True
        self.set_active(active)

    def set_active(self, active):
        self.background_color = BG_ACTIVE if active else BG_SLOT_ALT


class FileMenuItem(Button):
    def __init__(self, text='', **kwargs):
        kwargs.setdefault('size_hint_y', None)
        kwargs.setdefault('height', MENU_ITEM_H)
        kwargs.setdefault('size_hint_x', None)
        kwargs.setdefault('width', MENU_W)
        super().__init__(text=text, **kwargs)
        self.background_normal = ''
        self.background_down = ''
        self.border = (0, 0, 0, 0)
        self.background_color = BG_MENU_ITEM
        self.color = FG_MAIN
        self.font_size = '9sp'
        self.bold = True


class MainSlotButton(Button):
    def __init__(self, caption='', icon_kind=None, **kwargs):
        kwargs.setdefault('size_hint', (None, None))
        kwargs.setdefault('size', (SLOT_W, SLOT_H))
        super().__init__(text=caption, **kwargs)
        self.icon_kind = icon_kind
        self.background_normal = ''
        self.background_down = ''
        self.border = (0, 0, 0, 0)
        self.background_color = BG_SLOT
        self.color = FG_DIM
        self.font_size = '9.5sp'
        self.bold = True
        self.halign = 'center'
        self.valign = 'middle'
        self.bind(size=self._sync_text)
        self.bind(pos=self._update_arrow, size=self._update_arrow)
        self.bind(pos=self._update_icon, size=self._update_icon)

        with self.canvas.after:
            self._icon_color = Color(*ICON_SAT)
            self._icon_body = RoundedRectangle(pos=(0, 0), size=(0, 0), radius=[dp(1.5)])
            self._icon_panel_l = Rectangle(pos=(0, 0), size=(0, 0))
            self._icon_panel_r = Rectangle(pos=(0, 0), size=(0, 0))
            self._icon_link_l = Line(points=[], width=1.1)
            self._icon_link_r = Line(points=[], width=1.1)
            self._icon_dish = Triangle(points=[0, 0, 0, 0, 0, 0])
            Color(*BLACK)
            self._arrow = Triangle(points=[0, 0, 0, 0, 0, 0])

        self._sync_text()
        self._update_icon()
        self._update_arrow()

    def _sync_text(self, *_):
        self.text_size = self.size

    def _update_icon(self, *_):
        if self.icon_kind != 'satellite':
            self._icon_color.rgba = (0, 0, 0, 0)
            self._icon_body.size = (0, 0)
            self._icon_panel_l.size = (0, 0)
            self._icon_panel_r.size = (0, 0)
            self._icon_link_l.points = []
            self._icon_link_r.points = []
            self._icon_dish.points = [0, 0, 0, 0, 0, 0]
            return

        self._icon_color.rgba = ICON_SAT
        panel_w = dp(7)
        panel_h = dp(14)
        gap = dp(3)
        body_w = dp(12)
        body_h = dp(12)
        total_w = panel_w + gap + body_w + gap + panel_w
        start_x = self.center_x - (total_w / 2)
        body_x = start_x + panel_w + gap
        body_y = self.center_y - (body_h / 2) + dp(1)

        panel_l_x = start_x
        panel_l_y = self.center_y - (panel_h / 2) + dp(1)
        panel_r_x = body_x + body_w + gap
        panel_r_y = panel_l_y
        body_cy = body_y + (body_h / 2)
        panel_cy = panel_l_y + (panel_h / 2)

        self._icon_panel_l.pos = (panel_l_x, panel_l_y)
        self._icon_panel_l.size = (panel_w, panel_h)
        self._icon_panel_r.pos = (panel_r_x, panel_r_y)
        self._icon_panel_r.size = (panel_w, panel_h)
        self._icon_body.pos = (body_x, body_y)
        self._icon_body.size = (body_w, body_h)
        self._icon_link_l.points = [panel_l_x + panel_w, panel_cy, body_x, body_cy]
        self._icon_link_r.points = [body_x + body_w, body_cy, panel_r_x, panel_cy]
        dish_tip_x = body_x + body_w - dp(1)
        dish_tip_y = body_y - dp(2)
        self._icon_dish.points = [
            dish_tip_x - dp(8), dish_tip_y - dp(4),
            dish_tip_x, dish_tip_y,
            dish_tip_x - dp(4), dish_tip_y + dp(5),
        ]

    def _update_arrow(self, *_):
        s = dp(8)
        inset = dp(6)
        x2 = self.right - inset
        y1 = self.y + inset
        self._arrow.points = [x2 - s, y1, x2, y1, x2, y1 + s]

    def set_active(self, active):
        self.background_color = BG_SLOT_ALT if active else BG_SLOT
        self.color = FG_MAIN if active else FG_DIM


class SubSlotButton(Button):
    def __init__(self, text='', icon_kind=None, **kwargs):
        kwargs.setdefault('size_hint', (None, None))
        kwargs.setdefault('size', (SUB_SLOT_W, SUB_SLOT_H))
        super().__init__(text=text, **kwargs)
        self.icon_kind = icon_kind
        self.background_normal = ''
        self.background_down = ''
        self.border = (0, 0, 0, 0)
        self.background_color = BG_SUB_SLOT
        self.color = FG_MAIN
        self.font_size = '8sp'
        self.bold = True
        self.halign = 'center'
        self.valign = 'middle'
        self.bind(size=self._sync_text)
        self.bind(pos=self._update_icon, size=self._update_icon)

        with self.canvas.after:
            self._icon_color = Color(*LINE_ACCENT)
            self._icon_ring = Line(ellipse=(0, 0, 0, 0), width=1.2)
            self._icon_h = Line(points=[], width=1.1)
            self._icon_v = Line(points=[], width=1.1)
            self._icon_dot = Ellipse(pos=(0, 0), size=(0, 0))

        self._sync_text()
        self._update_icon()

    def _sync_text(self, *_):
        self.text_size = self.size

    def set_spec(self, text='', icon_kind=None):
        self.text = text
        self.icon_kind = icon_kind
        self._sync_text()
        self._update_icon()

    def _update_icon(self, *_):
        if self.icon_kind != 'my_location':
            self._icon_color.rgba = (0, 0, 0, 0)
            self._icon_ring.ellipse = (0, 0, 0, 0)
            self._icon_h.points = []
            self._icon_v.points = []
            self._icon_dot.pos = (0, 0)
            self._icon_dot.size = (0, 0)
            return

        self._icon_color.rgba = LINE_ACCENT
        cx = self.center_x
        cy = self.center_y
        r = dp(8)
        arm = dp(4)
        dot = dp(3.2)
        self._icon_ring.ellipse = (cx - r, cy - r, r * 2, r * 2)
        self._icon_h.points = [cx - r - arm, cy, cx - r + dp(1), cy, cx + r - dp(1), cy, cx + r + arm, cy]
        self._icon_v.points = [cx, cy - r - arm, cx, cy - r + dp(1), cx, cy + r - dp(1), cx, cy + r + arm]
        self._icon_dot.pos = (cx - dot / 2, cy - dot / 2)
        self._icon_dot.size = (dot, dot)


class TileMapView(StencilView):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.center_lat = None
        self.center_lon = None
        self.home_lat = None
        self.home_lon = None
        self.zoom = GPS_PARCEL_ZOOM
        self.style_key = 'satellite'
        self.on_view_changed = None

        self._center_world_x = None
        self._center_world_y = None
        self._tile_request_id = 0
        self._fetch_event = None
        self._worker_busy = False
        self._worker_pending = False
        self._base_tiles = {}
        self._overlay_tiles = {}
        self._visible_keys = []
        self._active_touches = {}
        self._drag_touch_uid = None
        self._pinch_base_distance = None
        self._pinch_base_zoom = float(self.zoom)
        self._preview_scale = 1.0
        self._grid_spacing_m = None
        self._meters_per_px = None
        self._utm_info = None

        with self.canvas.before:
            Color(*BG_MAP_VIEW)
            self._bg = RoundedRectangle(pos=self.pos, size=self.size, radius=[dp(10)])
        with self.canvas.after:
            self._grid_color = Color(0.22, 0.70, 1.00, 0.22)
            self._grid_lines = Line(points=[], width=1.0)
            self._cross_shadow_color = Color(0.02, 0.03, 0.05, 0.86)
            self._cross_shadow = Line(points=[], width=3.2)
            self._cross_color = Color(0.22, 0.70, 1.00, 0.94)
            self._cross = Line(points=[], width=1.55)
            self._marker_color = Color(*MYLOC_COLOR)
            self._marker_ring = Line(ellipse=(0, 0, 0, 0), width=1.8)
            self._marker_h = Line(points=[], width=1.2)
            self._marker_v = Line(points=[], width=1.2)
            self._marker_dot = Ellipse(pos=(0, 0), size=(0, 0))

        self.tile_layer = FloatLayout(size_hint=(None, None))
        self.add_widget(self.tile_layer)

        self.loading_label = Label(
            text='SATELLITE MAP READY',
            color=FG_MAIN,
            font_size='9sp',
            bold=True,
            halign='center',
            valign='middle',
            size_hint=(None, None),
        )
        self.loading_label.bind(size=lambda w, s: setattr(w, 'text_size', s))
        self.add_widget(self.loading_label)

        self.zoom_in_btn = Button(text='+', size_hint=(None, None), size=(dp(56), dp(56)))
        self.zoom_out_btn = Button(text='-', size_hint=(None, None), size=(dp(56), dp(56)))
        self.home_btn = Button(text='◎', size_hint=(None, None), size=(dp(56), dp(56)))
        for btn in (self.zoom_in_btn, self.zoom_out_btn, self.home_btn):
            btn.background_normal = ''
            btn.background_down = ''
            btn.border = (0, 0, 0, 0)
            btn.background_color = (0.10, 0.14, 0.20, 0.95)
            btn.color = FG_MAIN
            btn.bold = True
            btn.font_size = '22sp'
            self.add_widget(btn)
        self.zoom_in_btn.bind(on_release=lambda *_: self.change_zoom(+1))
        self.zoom_out_btn.bind(on_release=lambda *_: self.change_zoom(-1))
        self.home_btn.bind(on_release=lambda *_: self.go_home())
        for btn in (self.zoom_in_btn, self.zoom_out_btn, self.home_btn):
            btn.opacity = 0.0
            btn.disabled = True

        self.hud_label = Label(
            text='UTM --',
            color=FG_MAIN,
            font_size='7sp',
            bold=True,
            halign='left',
            valign='middle',
            size_hint=(None, None),
        )
        self.hud_label.bind(size=lambda w, s: setattr(w, 'text_size', s))
        self.add_widget(self.hud_label)

        self.east_labels = []
        self.north_labels = []
        for _ in range(6):
            lbl = Label(color=FG_MAIN, font_size='6.7sp', bold=True, halign='center', valign='middle', size_hint=(None, None), opacity=0)
            lbl.bind(size=lambda w, s: setattr(w, 'text_size', s))
            self.east_labels.append(lbl)
            self.add_widget(lbl)
        for _ in range(6):
            lbl = Label(color=FG_MAIN, font_size='6.7sp', bold=True, halign='left', valign='middle', size_hint=(None, None), opacity=0)
            lbl.bind(size=lambda w, s: setattr(w, 'text_size', s))
            self.north_labels.append(lbl)
            self.add_widget(lbl)

        self.bind(pos=self._layout_tiles, size=self._on_view_changed_size)
        self._layout_tiles()

    def _on_view_changed_size(self, *_):
        self._layout_tiles()
        if self.center_lat is not None and self.center_lon is not None:
            self._schedule_fetch(0.02)

    def set_style(self, style_key):
        style_key = style_key if style_key in MAP_STYLE_DEFS else 'satellite'
        if style_key == self.style_key:
            return
        self.style_key = style_key
        if self.center_lat is not None and self.center_lon is not None:
            self._schedule_fetch(0.01)

    def set_zoom_level(self, zoom, refresh=True):
        try:
            zoom = int(round(float(zoom)))
        except Exception:
            return
        zoom = max(TILE_MIN_ZOOM, min(TILE_MAX_ZOOM, zoom))
        if zoom == self.zoom:
            return
        self.zoom = zoom
        self._preview_scale = 1.0
        if self.center_lat is not None and self.center_lon is not None:
            self._center_world_x, self._center_world_y = latlon_to_world_px(self.center_lat, self.center_lon, self.zoom)
            if refresh:
                self._schedule_fetch(0.01)
            self._layout_tiles()
            self._notify_view_changed()

    def set_home_location(self, lat=None, lon=None):
        if lat is None or lon is None:
            return
        self.home_lat = float(lat)
        self.home_lon = float(lon)

    def go_home(self):
        if self.home_lat is None or self.home_lon is None:
            return
        self.set_location(self.home_lat, self.home_lon, reset_zoom=True, target_zoom=GPS_PARCEL_ZOOM)

    def set_location(self, lat=None, lon=None, reset_zoom=False, target_zoom=None):
        if lat is None or lon is None:
            self.center_lat = None
            self.center_lon = None
            self._center_world_x = None
            self._center_world_y = None
            self.clear_tiles('NO MAP LOCATION')
            return
        self.center_lat = float(lat)
        self.center_lon = float(lon)
        self.set_home_location(lat, lon)
        if target_zoom is not None:
            self.zoom = max(TILE_MIN_ZOOM, min(TILE_MAX_ZOOM, float(target_zoom)))
        elif reset_zoom:
            self.zoom = GPS_PARCEL_ZOOM
        self._preview_scale = 1.0
        self._center_world_x, self._center_world_y = latlon_to_world_px(self.center_lat, self.center_lon, self.zoom)
        self._layout_tiles()
        self._schedule_fetch(0.01)
        self._notify_view_changed()

    def change_zoom(self, delta):
        self.set_zoom_level(self.zoom + float(delta))

    def clear_tiles(self, message='NO MAP YET'):
        self._tile_request_id += 1
        self._visible_keys = []
        self._preview_scale = 1.0
        self._grid_lines.points = []
        self._cross.points = []
        self._cross_shadow.points = []
        self._grid_spacing_m = None
        self._meters_per_px = None
        self._utm_info = None
        if self._fetch_event is not None:
            try:
                self._fetch_event.cancel()
            except Exception:
                pass
            self._fetch_event = None
        for key, widget in list(self._base_tiles.items()):
            try:
                self.tile_layer.remove_widget(widget)
            except Exception:
                pass
        for key, widget in list(self._overlay_tiles.items()):
            try:
                self.tile_layer.remove_widget(widget)
            except Exception:
                pass
        self._base_tiles.clear()
        self._overlay_tiles.clear()
        self.loading_label.text = message
        self.loading_label.opacity = 1
        self._marker_color.rgba = (0, 0, 0, 0)
        for lbl in self.east_labels + self.north_labels:
            lbl.opacity = 0
            lbl.text = ''
        self.hud_label.text = 'UTM --'
        self._layout_tiles()

    def _schedule_fetch(self, delay=0.08):
        if self._fetch_event is not None:
            try:
                self._fetch_event.cancel()
            except Exception:
                pass
        self._fetch_event = Clock.schedule_once(self._begin_fetch, delay)

    def _source_zoom_for_style(self, style_key=None, view_zoom=None):
        try:
            view_zoom = int(round(float(self.zoom if view_zoom is None else view_zoom)))
        except Exception:
            view_zoom = int(round(float(self.zoom)))
        style = map_style_def(style_key or self.style_key)
        try:
            max_source_zoom = int(round(float(style.get('max_source_zoom', view_zoom))))
        except Exception:
            max_source_zoom = view_zoom
        return max(TILE_MIN_ZOOM, min(view_zoom, max_source_zoom))

    def _compute_visible_tile_specs(self, style_key=None, view_zoom=None):
        if self._center_world_x is None or self._center_world_y is None:
            self._center_world_x, self._center_world_y = latlon_to_world_px(self.center_lat, self.center_lon, self.zoom)
        try:
            view_zoom = int(round(float(self.zoom if view_zoom is None else view_zoom)))
        except Exception:
            view_zoom = int(round(float(self.zoom)))
        source_zoom = self._source_zoom_for_style(style_key or self.style_key, view_zoom)
        zoom_factor = 2 ** max(0, view_zoom - source_zoom)
        tile_world_size = 256.0 * zoom_factor
        center_x = self._center_world_x
        center_y = self._center_world_y
        pad = max(256.0, tile_world_size)
        left = center_x - (self.width / 2.0) - pad
        top = center_y - (self.height / 2.0) - pad
        right = center_x + (self.width / 2.0) + pad
        bottom = center_y + (self.height / 2.0) + pad
        tile_left = int(math.floor(left / tile_world_size))
        tile_top = int(math.floor(top / tile_world_size))
        tile_right = int(math.floor(right / tile_world_size))
        tile_bottom = int(math.floor(bottom / tile_world_size))
        max_tile = 2 ** int(source_zoom)
        specs = []
        for ty in range(tile_top, tile_bottom + 1):
            if ty < 0 or ty >= max_tile:
                continue
            for ux in range(tile_left, tile_right + 1):
                tx = ux % max_tile
                specs.append({'ux': ux, 'tx': tx, 'ty': ty, 'source_zoom': int(source_zoom)})
        return specs

    def _begin_fetch(self, *_):
        self._fetch_event = None
        if self.center_lat is None or self.center_lon is None or self.width <= 0 or self.height <= 0:
            return
        if self._worker_busy:
            self._worker_pending = True
            return
        specs = self._compute_visible_tile_specs(self.style_key, self.zoom)
        self._tile_request_id += 1
        request_id = self._tile_request_id
        self._worker_busy = True
        self.loading_label.text = 'LOADING MAP TILES...'
        self.loading_label.opacity = 1
        worker = threading.Thread(target=self._download_tiles_worker, args=(request_id, self.zoom, self.style_key, specs), daemon=True)
        worker.start()

    def _download_tiles_worker(self, request_id, zoom, style_key, specs):
        style = map_style_def(style_key)
        tile_records = []
        errors = []
        for spec in specs:
            tx = spec['tx']
            ty = spec['ty']
            source_zoom = int(spec.get('source_zoom', int(round(zoom))))
            base_cache = map_tile_cache_path(style['base_name'], source_zoom, tx, ty, style['base_ext'])
            base_url = render_tile_request_url(style['base_url'], source_zoom, tx, ty)
            base_path, base_error = download_map_tile(base_url, base_cache)
            if base_error:
                errors.append(f'base {tx}/{ty}: {base_error}')
            overlay_path = ''
            if style.get('overlay_url'):
                overlay_cache = map_tile_cache_path(style['overlay_name'], source_zoom, tx, ty, style['overlay_ext'])
                overlay_url = render_tile_request_url(style['overlay_url'], source_zoom, tx, ty)
                overlay_path, overlay_error = download_map_tile(overlay_url, overlay_cache)
                overlay_path = overlay_path or ''
                if overlay_error:
                    errors.append(f'overlay {tx}/{ty}: {overlay_error}')
            tile_records.append({
                'source_zoom': source_zoom,
                'ux': spec['ux'],
                'ty': ty,
                'base_path': base_path or '',
                'overlay_path': overlay_path,
            })
        payload = {
            'request_id': request_id,
            'zoom': zoom,
            'style_key': style_key,
            'tile_records': tile_records,
            'overlay_opacity': style.get('overlay_opacity', 0.0),
            'error_text': '\n'.join(errors[:4]),
            'ok': any(bool(item['base_path']) for item in tile_records),
        }
        Clock.schedule_once(lambda dt: self._apply_tile_payload(payload), 0)

    def _ensure_tile_widget(self, bucket, key):
        widget = bucket.get(key)
        if widget is not None:
            return widget
        widget = Image(size_hint=(None, None), allow_stretch=True, keep_ratio=False, opacity=0.0)
        bucket[key] = widget
        self.tile_layer.add_widget(widget)
        return widget

    def _apply_tile_payload(self, payload):
        self._worker_busy = False
        try:
            if payload.get('request_id') != self._tile_request_id:
                return
            keep_keys = []
            overlay_opacity = payload.get('overlay_opacity', 0.0)
            for item in payload.get('tile_records', []):
                key = (int(item.get('source_zoom', round(payload.get('zoom', self.zoom)))), item['ux'], item['ty'])
                keep_keys.append(key)
                base_widget = self._ensure_tile_widget(self._base_tiles, key)
                overlay_widget = self._ensure_tile_widget(self._overlay_tiles, key)
                if base_widget.source != (item['base_path'] or ''):
                    base_widget.source = item['base_path'] or ''
                    if base_widget.source:
                        try:
                            base_widget.reload()
                        except Exception:
                            pass
                base_widget.opacity = 1.0 if item['base_path'] else 0.0
                if overlay_widget.source != (item['overlay_path'] or ''):
                    overlay_widget.source = item['overlay_path'] or ''
                    if overlay_widget.source:
                        try:
                            overlay_widget.reload()
                        except Exception:
                            pass
                overlay_widget.opacity = overlay_opacity if item['overlay_path'] else 0.0

            keep_set = set(keep_keys)
            for key, widget in list(self._base_tiles.items()):
                if key not in keep_set:
                    try:
                        self.tile_layer.remove_widget(widget)
                    except Exception:
                        pass
                    self._base_tiles.pop(key, None)
            for key, widget in list(self._overlay_tiles.items()):
                if key not in keep_set:
                    try:
                        self.tile_layer.remove_widget(widget)
                    except Exception:
                        pass
                    self._overlay_tiles.pop(key, None)

            self._visible_keys = keep_keys
            if payload.get('ok'):
                self.loading_label.opacity = 0.0
                self._marker_color.rgba = MYLOC_COLOR
            else:
                self.loading_label.text = 'MAP TILES NOT READY'
                if payload.get('error_text'):
                    self.loading_label.text = f"MAP DOWNLOAD FAILED\n{payload['error_text']}"
                self.loading_label.opacity = 1.0
            self._layout_tiles()
            self._notify_view_changed()
        finally:
            if self._worker_pending:
                self._worker_pending = False
                self._schedule_fetch(0.02)

    def _notify_view_changed(self):
        if callable(self.on_view_changed) and self.center_lat is not None and self.center_lon is not None:
            try:
                self.on_view_changed(self.center_lat, self.center_lon, self.zoom)
            except Exception:
                pass

    def _visible_key_set(self):
        return set(self._visible_keys)

    def _current_needed_key_set(self):
        return set((int(spec.get('source_zoom', round(self.zoom))), spec['ux'], spec['ty']) for spec in self._compute_visible_tile_specs(self.style_key, self.zoom))

    def _needs_fetch_for_current_view(self):
        if self.center_lat is None or self.center_lon is None:
            return False
        needed = self._current_needed_key_set()
        if not needed:
            return False
        return not needed.issubset(self._visible_key_set())

    def on_touch_down(self, touch):
        handled = super().on_touch_down(touch)
        if handled:
            return True
        if not self.collide_point(*touch.pos):
            return False
        if self.center_lat is None or self.center_lon is None:
            return False
        self._active_touches[touch.uid] = (touch.x, touch.y)
        touch.grab(self)
        if len(self._active_touches) == 1:
            self._drag_touch_uid = touch.uid
        elif len(self._active_touches) >= 2:
            self._pinch_base_distance = self._two_touch_distance()
            self._pinch_base_zoom = float(self.zoom)
        return True

    def on_touch_move(self, touch):
        if touch.grab_current is not self:
            return super().on_touch_move(touch)
        prev = self._active_touches.get(touch.uid, (touch.x, touch.y))
        self._active_touches[touch.uid] = (touch.x, touch.y)

        if len(self._active_touches) >= 2:
            distance = self._two_touch_distance()
            if distance is not None:
                if not self._pinch_base_distance or self._pinch_base_distance <= 0:
                    self._pinch_base_distance = distance
                scale = max(0.5, min(2.2, distance / max(1.0, self._pinch_base_distance)))
                self._preview_scale = scale
                self._layout_tiles()
            return True

        dx = touch.x - prev[0]
        dy = touch.y - prev[1]
        if self._center_world_x is None or self._center_world_y is None:
            self._center_world_x, self._center_world_y = latlon_to_world_px(self.center_lat, self.center_lon, self.zoom)
        self._center_world_x -= dx
        self._center_world_y -= dy
        self.center_lat, self.center_lon = world_px_to_latlon(self._center_world_x, self._center_world_y, self.zoom)
        self._layout_tiles()
        self._notify_view_changed()
        if self._needs_fetch_for_current_view() and not self._worker_busy:
            self._schedule_fetch(0.03)
        return True

    def on_touch_up(self, touch):
        if touch.grab_current is self:
            touch.ungrab(self)
            self._active_touches.pop(touch.uid, None)
            if self._pinch_base_distance and len(self._active_touches) < 2:
                scale = max(0.5, min(2.2, self._preview_scale))
                preview_zoom = self._pinch_base_zoom + math.log(scale, 2)
                self._pinch_base_distance = None
                self._preview_scale = 1.0
                self.set_zoom_level(preview_zoom, refresh=True)
                return True
            self._preview_scale = 1.0
            self._layout_tiles()
            if self._needs_fetch_for_current_view():
                self._schedule_fetch(0.02)
            return True
        return super().on_touch_up(touch)

    def _two_touch_distance(self):
        if len(self._active_touches) < 2:
            return None
        pts = list(self._active_touches.values())[:2]
        (x1, y1), (x2, y2) = pts
        return math.hypot(x2 - x1, y2 - y1)

    def _tile_geometry_for_current_zoom(self, z_value, ux, ty, world_left, world_top):
        try:
            z_value = float(z_value)
        except Exception:
            z_value = float(self.zoom)
        zoom_delta = float(self.zoom) - z_value
        zoom_factor = 2.0 ** zoom_delta
        tile_world = 256.0 * zoom_factor
        sx = self.x + ((ux * tile_world) - world_left) * self._preview_scale
        sy_top = ((ty * tile_world) - world_top) * self._preview_scale
        tile_size = tile_world * self._preview_scale
        sy = self.top - sy_top - tile_size
        return sx, sy, tile_size

    def _layout_tiles(self, *_):
        self._bg.pos = self.pos
        self._bg.size = self.size
        self.tile_layer.pos = self.pos
        self.tile_layer.size = self.size

        if self._center_world_x is None or self._center_world_y is None:
            if self.center_lat is not None and self.center_lon is not None:
                self._center_world_x, self._center_world_y = latlon_to_world_px(self.center_lat, self.center_lon, self.zoom)
            else:
                self.loading_label.size = (min(self.width - dp(12), dp(260)), dp(44))
                self.loading_label.pos = (self.x + (self.width - self.loading_label.width) / 2.0, self.y + (self.height - self.loading_label.height) / 2.0)
                return

        world_left = self._center_world_x - (self.width / 2.0)
        world_top = self._center_world_y - (self.height / 2.0)
        tile_scale = 256.0 * self._preview_scale

        for key, widget in self._base_tiles.items():
            z_value, ux, ty = key
            sx, sy, tile_size = self._tile_geometry_for_current_zoom(z_value, ux, ty, world_left, world_top)
            widget.pos = (sx, sy)
            widget.size = (tile_size + 1.0, tile_size + 1.0)
        for key, widget in self._overlay_tiles.items():
            z_value, ux, ty = key
            sx, sy, tile_size = self._tile_geometry_for_current_zoom(z_value, ux, ty, world_left, world_top)
            widget.pos = (sx, sy)
            widget.size = (tile_size + 1.0, tile_size + 1.0)

        self.loading_label.size = (min(max(dp(180), self.width - dp(14)), dp(360)), dp(44))
        self.loading_label.pos = (self.x + (self.width - self.loading_label.width) / 2.0, self.y + (self.height - self.loading_label.height) / 2.0)

        btn_x = self.right - dp(62)
        self.zoom_in_btn.pos = (btn_x, self.top - dp(66))
        self.zoom_out_btn.pos = (btn_x, self.top - dp(128))
        self.home_btn.pos = (btn_x, self.top - dp(190))
        self.hud_label.size = (min(dp(330), max(dp(120), self.width - dp(146))), dp(16))
        self.hud_label.pos = (self.x + dp(8), self.top - dp(22))

        cx = self.center_x
        cy = self.center_y
        r = min(self.width, self.height) * 0.038
        arm = r * 0.50
        dot = max(dp(4), r * 0.44)
        self._cross_shadow.points = [cx - dp(18), cy, cx + dp(18), cy, cx, cy - dp(18), cx, cy + dp(18)]
        self._cross.points = [cx - dp(16), cy, cx + dp(16), cy, cx, cy - dp(16), cx, cy + dp(16)]
        self._marker_ring.ellipse = (cx - r, cy - r, r * 2.0, r * 2.0)
        self._marker_h.points = [cx - r - arm, cy, cx - r + dp(1), cy, cx + r - dp(1), cy, cx + r + arm, cy]
        self._marker_v.points = [cx, cy - r - arm, cx, cy - r + dp(1), cx, cy + r - dp(1), cx, cy + r + arm]
        self._marker_dot.pos = (cx - dot / 2.0, cy - dot / 2.0)
        self._marker_dot.size = (dot, dot)

        self._update_grid_overlay()

    def _update_grid_overlay(self):
        if self.center_lat is None or self.center_lon is None:
            self._grid_lines.points = []
            self._cross.points = []
            self._cross_shadow.points = []
            self._grid_spacing_m = None
            self._meters_per_px = None
            self._utm_info = None
            for lbl in self.east_labels + self.north_labels:
                lbl.opacity = 0
                lbl.text = ''
            self.hud_label.text = 'UTM --'
            return

        try:
            utm = latlon_to_utm(self.center_lat, self.center_lon)
        except Exception:
            self._grid_lines.points = []
            self._grid_spacing_m = None
            self._meters_per_px = None
            self._utm_info = None
            self.hud_label.text = 'UTM unavailable'
            return

        self._utm_info = utm
        base_mpp = 156543.03392804097 * math.cos(math.radians(self.center_lat)) / (2 ** self.zoom)
        meters_per_px = max(0.01, base_mpp / max(0.35, self._preview_scale))
        self._meters_per_px = meters_per_px
        spacing = choose_grid_spacing(meters_per_px)
        self._grid_spacing_m = spacing

        e0 = utm['easting']
        n0 = utm['northing']
        half_w_m = meters_per_px * self.width / 2.0
        half_h_m = meters_per_px * self.height / 2.0

        points = []
        verticals = []
        horizontals = []

        start_e = math.floor((e0 - half_w_m) / spacing) * spacing
        end_e = math.ceil((e0 + half_w_m) / spacing) * spacing
        e = start_e
        while e <= end_e + 0.001:
            px = self.center_x + ((e - e0) / meters_per_px)
            if self.x <= px <= self.right:
                points.extend([px, self.y, px, self.top])
                verticals.append((px, e))
            e += spacing

        start_n = math.floor((n0 - half_h_m) / spacing) * spacing
        end_n = math.ceil((n0 + half_h_m) / spacing) * spacing
        n = start_n
        while n <= end_n + 0.001:
            py = self.center_y + ((n - n0) / meters_per_px)
            if self.y <= py <= self.top:
                points.extend([self.x, py, self.right, py])
                horizontals.append((py, n))
            n += spacing

        self._grid_lines.points = points
        self.hud_label.text = f"UTM Z{utm['zone']}{utm['band']} | z{self.zoom:.2f} | {meters_per_px:.2f} m/px | grid {spacing} m"

        for lbl in self.east_labels + self.north_labels:
            lbl.opacity = 0
            lbl.text = ''

        visible_verticals = [item for item in verticals if (self.x + dp(24)) <= item[0] <= (self.right - dp(24))]
        visible_horizontals = [item for item in horizontals if (self.y + dp(24)) <= item[0] <= (self.top - dp(24))]

        for lbl, (px, e) in zip(self.east_labels, visible_verticals[:len(self.east_labels)]):
            lbl.text = f"E {format_utm_meters(e)}"
            lbl.size = (dp(78), dp(14))
            lx = max(self.x + dp(4), min(px - lbl.width / 2.0, self.right - lbl.width - dp(4)))
            lbl.pos = (lx, self.y + dp(3))
            lbl.opacity = 0.92

        for lbl, (py, n) in zip(self.north_labels, visible_horizontals[:len(self.north_labels)]):
            lbl.text = f"N {format_utm_meters(n)}"
            lbl.size = (dp(90), dp(14))
            ly = max(self.y + dp(4), min(py - lbl.height / 2.0, self.top - lbl.height - dp(4)))
            lbl.pos = (self.x + dp(5), ly)
            lbl.opacity = 0.92


class MapPanel(FloatLayout):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.compact_mode = False
        self.current_style_key = 'satellite'
        self.current_provider = ''
        self.last_probe_text = '-'
        self.last_note_text = 'PyStation-style internal map restored for Android/Pydroid tests. Drag, pinch, +/-, and ◎ recenter are active inside the panel.'
        self._ignore_view_callback = False

        with self.canvas.before:
            Color(0.02, 0.03, 0.05, 0.34)
            self._shadow = RoundedRectangle(pos=(0, 0), size=(0, 0), radius=[dp(18)])
            Color(0.15, 0.19, 0.25, 0.985)
            self._bg = RoundedRectangle(pos=self.pos, size=self.size, radius=[dp(18)])
            Color(0.22, 0.70, 1.00, 0.65)
            self._border = Line(rounded_rectangle=(0, 0, 0, 0, dp(18)), width=1.4)
        self.bind(pos=self._layout_panel, size=self._layout_panel)

        self.title_label = Label(text='GPS / UTM MAP', color=FG_MAIN, bold=True, font_size='13sp', size_hint=(None, None))
        self.source_label = Label(text='SOURCE: MY LOCATION', color=LINE_ACCENT, font_size='10sp', bold=True, size_hint=(None, None))
        self.style_label = Label(text='STYLE: SATELLITE', color=FG_DIM, font_size='8.5sp', size_hint=(None, None), shorten=True, shorten_from='right')
        self.coord_label = Label(text='LAT: --   LON: --', color=FG_MAIN, font_size='8.4sp', size_hint=(None, None), shorten=True, shorten_from='right')
        self.utm_label = Label(text='UTM: --', color=FG_MAIN, font_size='8.2sp', size_hint=(None, None), shorten=True, shorten_from='right')
        self.probe_label = Label(text='PROBE: -', color=FG_DIM, font_size='7.2sp', size_hint=(None, None), shorten=True, shorten_from='right')
        self.note_label = Label(text=self.last_note_text, color=FG_DIM, font_size='7.2sp', size_hint=(None, None), shorten=True, shorten_from='right')

        for lbl in (self.title_label, self.source_label, self.style_label, self.coord_label, self.utm_label, self.probe_label, self.note_label):
            lbl.halign = 'left'
            lbl.valign = 'middle'
            lbl.bind(size=lambda w, s: setattr(w, 'text_size', s))
            self.add_widget(lbl)

        self.view = TileMapView(size_hint=(None, None))
        self.view.on_view_changed = self._handle_view_changed
        self.add_widget(self.view)

        self.zoom_in_btn = Button(text='+', size_hint=(None, None), size=(dp(56), dp(56)))
        self.zoom_out_btn = Button(text='-', size_hint=(None, None), size=(dp(56), dp(56)))
        self.home_btn = Button(text='◎', size_hint=(None, None), size=(dp(56), dp(56)))
        for btn in (self.zoom_in_btn, self.zoom_out_btn, self.home_btn):
            btn.background_normal = ''
            btn.background_down = ''
            btn.border = (0, 0, 0, 0)
            btn.background_color = (0.10, 0.14, 0.20, 0.96)
            btn.color = FG_MAIN
            btn.bold = True
            btn.font_size = '22sp'
            self.add_widget(btn)
        self.zoom_in_btn.bind(on_release=lambda *_: self.view.change_zoom(+1))
        self.zoom_out_btn.bind(on_release=lambda *_: self.view.change_zoom(-1))
        self.home_btn.bind(on_release=lambda *_: self.view.go_home())
        self._layout_panel()

    def set_compact_mode(self, compact):
        self.compact_mode = bool(compact)
        self._layout_panel()

    def set_context(self, source='MY LOCATION', style='SATELLITE'):
        self.source_label.text = f'SOURCE: {source}'
        self.style_label.text = f'STYLE: {style}'

    def set_map_style(self, style_key, style_label=None):
        self.current_style_key = style_key if style_key in MAP_STYLE_DEFS else 'satellite'
        self.view.set_style(self.current_style_key)
        style_info = map_style_def(self.current_style_key)
        self.style_label.text = f"STYLE: {style_label or style_info.get('label', 'MAP')}"
        if self.view.center_lat is not None and self.view.center_lon is not None:
            self.note_label.text = 'PyStation-style internal map active | drag, pinch, +/-, ◎ recenter'

    def set_zoom_level(self, zoom):
        self.view.set_zoom_level(zoom)

    def set_status(self, text):
        if text:
            self.note_label.text = text

    def set_shell_message(self, text):
        self.view.clear_tiles(text or 'MAP MESSAGE')

    def set_probe(self, text):
        self.last_probe_text = text or '-'
        probe_single = self.last_probe_text.replace('\n', ' | ')
        self.probe_label.text = f'PROBE: {probe_single}'

    def set_note(self, text):
        self.last_note_text = text or ''
        self.note_label.text = self.last_note_text

    def _set_coord_labels(self, lat=None, lon=None, provider=''):
        if lat is None or lon is None:
            self.coord_label.text = 'LAT: --   LON: --'
            self.utm_label.text = 'UTM: --'
            return
        provider_text = f'   PROVIDER: {provider}' if provider else ''
        self.coord_label.text = f'LAT: {lat:.6f}   LON: {lon:.6f}{provider_text}'
        try:
            utm = latlon_to_utm(lat, lon)
            grid_text = self.view._grid_spacing_m if self.view._grid_spacing_m else '--'
            self.utm_label.text = f"UTM: Z{utm['zone']}{utm['band']}{utm['hemisphere']}  E={utm['easting']:.2f}  N={utm['northing']:.2f}  GRID={grid_text}m"
        except Exception:
            self.utm_label.text = 'UTM: unavailable'

    def set_coordinates(self, lat=None, lon=None, provider='', reset_zoom=False, target_zoom=None):
        self.current_provider = provider or ''
        self._set_coord_labels(lat, lon, provider)
        self._ignore_view_callback = True
        try:
            self.view.set_location(lat, lon, reset_zoom=reset_zoom, target_zoom=target_zoom)
        finally:
            self._ignore_view_callback = False

    def _handle_view_changed(self, lat, lon, zoom):
        if self._ignore_view_callback:
            return
        self._set_coord_labels(lat, lon, self.current_provider)
        self.note_label.text = f'Kivy phone shell | fixed frame baseline | zoom ref {zoom}'

    def _layout_panel(self, *_):
        shadow_pad = dp(7)
        self._shadow.pos = (self.x + shadow_pad, self.y - shadow_pad)
        self._shadow.size = self.size
        self._bg.pos = self.pos
        self._bg.size = self.size
        self._border.rounded_rectangle = (self.x, self.y, self.width, self.height, dp(18))

        compact = self.compact_mode or self.width <= dp(430)
        pad_x = dp(10 if compact else 16)
        top_pad = dp(10 if compact else 14)
        bottom_pad = dp(8 if compact else 12)
        inner_w = max(dp(100), self.width - pad_x * 2)
        top = self.top - top_pad

        self.title_label.font_size = '11sp' if compact else '13sp'
        self.source_label.font_size = '9sp' if compact else '10sp'
        self.style_label.font_size = '7.5sp' if compact else '8.5sp'
        self.coord_label.font_size = '7.6sp' if compact else '8.4sp'
        self.utm_label.font_size = '7.3sp' if compact else '8.2sp'
        self.probe_label.font_size = '6.5sp' if compact else '7.2sp'
        self.note_label.font_size = '6.5sp' if compact else '7.2sp'

        title_h = dp(18 if compact else 20)
        row_h = dp(14 if compact else 16)
        note_h = dp(12 if compact else 14)
        step = dp(10 if compact else 12)

        self.title_label.size = (inner_w, title_h)
        self.title_label.pos = (self.x + pad_x, top - title_h)
        self.source_label.size = (inner_w, row_h)
        self.source_label.pos = (self.x + pad_x, self.title_label.y - step)
        self.style_label.size = (inner_w, row_h)
        self.style_label.pos = (self.x + pad_x, self.source_label.y - step)
        self.coord_label.size = (inner_w, row_h)
        self.coord_label.pos = (self.x + pad_x, self.style_label.y - step)
        self.utm_label.size = (inner_w, row_h)
        self.utm_label.pos = (self.x + pad_x, self.coord_label.y - step)
        self.probe_label.size = (inner_w, row_h)
        self.probe_label.pos = (self.x + pad_x, self.utm_label.y - step)
        self.note_label.size = (inner_w, note_h)
        self.note_label.pos = (self.x + pad_x, self.y + bottom_pad)

        view_y = self.note_label.top + dp(8 if compact else 10)
        view_top = self.probe_label.y - dp(8 if compact else 10)
        view_h = max(dp(150 if compact else 170), view_top - view_y)
        self.view.pos = (self.x + pad_x, view_y)
        self.view.size = (inner_w, view_h)

        btn_size = dp(54 if compact else 56)
        btn_gap = dp(8)
        btn_pad = dp(10)
        btn_x = self.view.right - btn_size - btn_pad
        top_y = self.view.top - btn_pad - btn_size
        for btn in (self.zoom_in_btn, self.zoom_out_btn, self.home_btn):
            btn.size = (btn_size, btn_size)
        self.zoom_in_btn.pos = (btn_x, top_y)
        self.zoom_out_btn.pos = (btn_x, top_y - btn_size - btn_gap)
        self.home_btn.pos = (btn_x, top_y - (btn_size + btn_gap) * 2)


class StatusBar(BoxLayout):
    def __init__(self, **kwargs):
        kwargs.setdefault('orientation', 'horizontal')
        kwargs.setdefault('size_hint_y', None)
        kwargs.setdefault('height', STATUS_H)
        kwargs.setdefault('padding', [dp(6), dp(4), dp(6), dp(4)])
        super().__init__(**kwargs)

        with self.canvas.before:
            Color(*BG_STATUS)
            self._bg = Rectangle(pos=self.pos, size=self.size)
            Color(*LINE_ACCENT)
            self._line = Rectangle(pos=(self.x, self.top - dp(1.2)), size=(self.width, dp(1.2)))
        self.bind(pos=self._sync_bg, size=self._sync_bg)

        self.caption = Label(
            text='STATUS',
            color=LINE_ACCENT,
            bold=True,
            font_size='9sp',
            size_hint=(None, 1),
            width=dp(58),
            halign='left',
            valign='middle',
        )
        self.caption.bind(size=lambda w, s: setattr(w, 'text_size', s))
        self.add_widget(self.caption)

        self.message = Label(
            text='READY',
            color=FG_MAIN,
            font_size='9sp',
            size_hint=(1, 1),
            halign='left',
            valign='middle',
            shorten=True,
            shorten_from='right',
        )
        self.message.bind(size=lambda w, s: setattr(w, 'text_size', s))
        self.add_widget(self.message)

    def set_message(self, text):
        self.message.text = text or 'READY'

    def _sync_bg(self, *_):
        self._bg.pos = self.pos
        self._bg.size = self.size
        self._line.pos = (self.x, self.top - dp(1.2))
        self._line.size = (self.width, dp(1.2))


class ProbePanel(BoxLayout):
    def __init__(self, **kwargs):
        kwargs.setdefault('orientation', 'vertical')
        kwargs.setdefault('size_hint', (None, None))
        kwargs.setdefault('spacing', dp(4))
        kwargs.setdefault('padding', [dp(10), dp(8), dp(10), dp(8)])
        super().__init__(**kwargs)

        with self.canvas.before:
            Color(*BG_MAP_CARD)
            self._bg = RoundedRectangle(pos=self.pos, size=self.size, radius=[dp(10)])
            Color(0.22, 0.70, 1.00, 0.35)
            self._border = Line(rounded_rectangle=(0, 0, 0, 0, dp(10)), width=1.0)
        self.bind(pos=self._sync_bg, size=self._sync_bg)

        self.title_label = Label(
            text='RUNTIME PROBE',
            color=LINE_ACCENT,
            bold=True,
            font_size='9.5sp',
            size_hint=(1, None),
            height=dp(16),
            halign='left',
            valign='middle',
        )
        self.title_label.bind(size=lambda w, s: setattr(w, 'text_size', s))
        self.add_widget(self.title_label)

        self.body_label = Label(
            text='tap GPS > MY LOCATION',
            color=FG_MAIN,
            font_size='7.4sp',
            size_hint=(1, 1),
            halign='left',
            valign='top',
        )
        self.body_label.bind(size=lambda w, s: setattr(w, 'text_size', s))
        self.add_widget(self.body_label)

    def set_text(self, text):
        self.body_label.text = text or '-'

    def _sync_bg(self, *_):
        self._bg.pos = self.pos
        self._bg.size = self.size
        self._border.rounded_rectangle = (self.x, self.y, self.width, self.height, dp(10))


class FileDropdown(DropDown):
    def __init__(self, on_pick=None, **kwargs):
        super().__init__(**kwargs)
        self.on_pick = on_pick
        self.auto_width = False
        self.auto_dismiss = False
        self.width = MENU_W
        self.max_height = dp(360)

        for item in FILE_MENU_ITEMS:
            btn = FileMenuItem(text=item)
            btn.bind(on_release=lambda _btn, label=item: self._pick(label))
            self.add_widget(btn)

    def _pick(self, label):
        if self.on_pick:
            self.on_pick(label)
        self.dismiss()


class InlineSubmenuRow(BoxLayout):
    def __init__(self, main_index=1, on_pick=None, **kwargs):
        kwargs.setdefault('orientation', 'horizontal')
        kwargs.setdefault('size_hint', (None, None))
        kwargs.setdefault('width', SUBMENU_W)
        kwargs.setdefault('height', SUBMENU_H)
        kwargs.setdefault('spacing', SUBMENU_SPACING)
        kwargs.setdefault('padding', [SUBMENU_PAD_X, (SLOT_H - SUB_SLOT_H) / 2, SUBMENU_PAD_X, (SLOT_H - SUB_SLOT_H) / 2])
        super().__init__(**kwargs)
        self.main_index = main_index
        self.on_pick = on_pick
        self.slot_buttons = []
        self.current_specs = []

        with self.canvas.before:
            Color(*BG_SUB_PANEL)
            self._bg = RoundedRectangle(pos=self.pos, size=self.size, radius=[dp(8)])
        self.bind(pos=self._sync_bg, size=self._sync_bg)

        for child_index in range(1, 6):
            btn = SubSlotButton(text='')
            btn.bind(on_release=lambda _btn, idx=child_index: self._emit_pick(idx))
            self.slot_buttons.append(btn)
            self.add_widget(btn)

        self.set_main_index(main_index)

    def set_main_index(self, main_index):
        self.main_index = main_index
        self.current_specs = submenu_specs_for_main(main_index)
        for btn, spec in zip(self.slot_buttons, self.current_specs):
            btn.set_spec(text=spec.get('text', ''), icon_kind=spec.get('icon_kind'))

    def _emit_pick(self, child_index):
        if not self.on_pick:
            return
        spec = {}
        if 1 <= child_index <= len(self.current_specs):
            spec = self.current_specs[child_index - 1]
        self.on_pick(self.main_index, child_index, spec)

    def _sync_bg(self, *_):
        self._bg.pos = self.pos
        self._bg.size = self.size


class MainTouchRow(BoxLayout):
    def __init__(self, caption='', main_index=1, on_toggle=None, on_submenu_pick=None, icon_kind=None, **kwargs):
        kwargs.setdefault('orientation', 'horizontal')
        kwargs.setdefault('size_hint', (None, None))
        kwargs.setdefault('width', ROW_W)
        kwargs.setdefault('height', SLOT_H)
        kwargs.setdefault('spacing', ROW_GAP_X)
        kwargs.setdefault('padding', [RAIL_PAD_X, 0, 0, 0])
        super().__init__(**kwargs)
        self.caption = caption
        self.main_index = main_index
        self.on_toggle = on_toggle
        self.on_submenu_pick = on_submenu_pick
        self.is_open = False
        self._tap_scheduled = False
        self._scheduled_open_state = False

        self.main_button = MainSlotButton(caption=caption, icon_kind=icon_kind)
        self.main_button.bind(on_press=lambda *_: self.queue_toggle())
        self.add_widget(self.main_button)

        self.submenu = InlineSubmenuRow(main_index=main_index, on_pick=self._handle_submenu_pick)
        self.submenu.opacity = 0
        self.submenu.disabled = True
        self.submenu.width = 0
        self.add_widget(self.submenu)

    def queue_toggle(self):
        if self._tap_scheduled:
            return
        self._tap_scheduled = True
        self._scheduled_open_state = not self.is_open
        Clock.schedule_once(self._emit_toggle, 0.02)

    def _emit_toggle(self, *_):
        self._tap_scheduled = False
        want_open = self._scheduled_open_state
        if self.on_toggle:
            self.on_toggle(self, want_open)
        else:
            self.set_open(want_open)

    def _handle_submenu_pick(self, main_index, child_index, spec):
        if self.on_submenu_pick:
            self.on_submenu_pick(main_index, child_index, spec)

    def set_open(self, is_open):
        self.is_open = is_open
        self.main_button.set_active(is_open)
        self.submenu.set_main_index(self.main_index)
        self.submenu.opacity = 1 if is_open else 0
        self.submenu.disabled = not is_open
        self.submenu.width = SUBMENU_W if is_open else 0
        self.do_layout()
        if self.parent:
            self.parent.do_layout()


class TopBar(BoxLayout):
    def __init__(self, status_callback=None, **kwargs):
        kwargs.setdefault('orientation', 'horizontal')
        kwargs.setdefault('size_hint_y', None)
        kwargs.setdefault('height', TOP_H)
        kwargs.setdefault('spacing', dp(4))
        kwargs.setdefault('padding', [dp(6), dp(6), dp(6), dp(6)])
        super().__init__(**kwargs)
        self.status_callback = status_callback
        self.file_is_open = False
        self._file_open_scheduled = False

        with self.canvas.before:
            Color(*BG_TOP)
            self._bg = Rectangle(pos=self.pos, size=self.size)
            Color(*LINE_ACCENT)
            self._line = Rectangle(pos=(self.x, self.y), size=(self.width, dp(1.4)))
        self.bind(pos=self._sync_bg, size=self._sync_bg)

        self.file_button = FlatMenuButton('FILE')
        self.edit_button = FlatMenuButton('EDIT')
        self.element_button = FlatMenuButton('ELEMENT')
        self.tools_button = FlatMenuButton('TOOLS', active=True)

        self.add_widget(self.file_button)
        self.add_widget(self.edit_button)
        self.add_widget(self.element_button)
        self.add_widget(self.tools_button)

        self.file_dropdown = FileDropdown(on_pick=self._on_file_pick)
        self.file_dropdown.bind(on_dismiss=lambda *_: self._set_file_open(False))
        self.file_button.bind(on_release=lambda *_: self.toggle_file_dropdown())

    def toggle_file_dropdown(self):
        if self.file_is_open or self._file_open_scheduled:
            self._file_open_scheduled = False
            self.file_dropdown.dismiss()
            self._set_file_open(False)
        else:
            self._file_open_scheduled = True
            Clock.schedule_once(self._open_file_dropdown, 0.05)

    def _open_file_dropdown(self, *_):
        self._file_open_scheduled = False
        self._set_file_open(True)
        self.file_dropdown.open(self.file_button)
        if self.status_callback:
            self.status_callback('FILE menu opened')

    def _set_file_open(self, is_open):
        self.file_is_open = is_open
        self.file_button.set_active(is_open)
        self.tools_button.set_active(not is_open)

    def _on_file_pick(self, label):
        self._set_file_open(False)
        if self.status_callback:
            self.status_callback(f'FILE > {label}')

    def _sync_bg(self, *_):
        self._bg.pos = self.pos
        self._bg.size = self.size
        self._line.pos = (self.x, self.y)
        self._line.size = (self.width, dp(1.4))


class WorkspaceHost(FloatLayout):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        with self.canvas.before:
            Color(*BG_WORK)
            self._bg = Rectangle(pos=self.pos, size=self.size)
        self.bind(pos=self._sync_bg, size=self._sync_bg)

        with self.canvas.after:
            Color(0.36, 0.43, 0.52, 0.16)
            self._chip = RoundedRectangle(pos=(0, 0), size=(0, 0), radius=[dp(8)])

        self.placeholder_label = Label(
            text='WORKSPACE',
            color=FG_SOFT,
            bold=False,
            font_size='10sp',
            size_hint=(None, None),
            size=(dp(92), dp(22)),
        )
        self.placeholder_label.bind(size=lambda w, s: setattr(w, 'text_size', s))
        self.add_widget(self.placeholder_label)

        self.map_panel = MapPanel(size_hint=(None, None))
        self.map_panel.opacity = 0
        self.map_panel.disabled = True
        self.add_widget(self.map_panel)

        self.current_style_key = 'satellite'
        self.current_zoom = GPS_PARCEL_ZOOM
        self.current_lat = None
        self.current_lon = None

        self.bind(pos=self._layout_items, size=self._layout_items)
        self._layout_items()

    def show_map_panel(self, source='MY LOCATION', style='SATELLITE + BOUNDARIES'):
        self.map_panel.set_context(source=source, style=style)
        self.map_panel.opacity = 1
        self.map_panel.disabled = False
        self.placeholder_label.opacity = 0
        self._layout_items()

    def set_map_status(self, text):
        self.map_panel.set_status(text)

    def set_map_shell_message(self, text):
        self.map_panel.set_shell_message(text)

    def set_map_probe(self, text):
        self.map_panel.set_probe(text)

    def set_map_note(self, text):
        self.map_panel.set_note(text)

    def set_map_coordinates(self, lat=None, lon=None, provider='', reset_zoom=False, target_zoom=None):
        self.map_panel.set_coordinates(lat, lon, provider, reset_zoom=reset_zoom, target_zoom=target_zoom)
        if lat is None or lon is None:
            return
        self.current_lat = float(lat)
        self.current_lon = float(lon)
        if target_zoom is not None:
            self.current_zoom = target_zoom

    def set_map_style(self, style_key, style_label=None):
        self.current_style_key = style_key if style_key in MAP_STYLE_DEFS else 'satellite'
        self.map_panel.set_map_style(self.current_style_key, style_label)

    def set_map_zoom(self, zoom):
        self.current_zoom = zoom
        self.map_panel.set_zoom_level(zoom)

    def _sync_bg(self, *_):
        self._bg.pos = self.pos
        self._bg.size = self.size
        self._layout_items()

    def _layout_items(self, *_):
        compact = self.width <= dp(430)
        margin = dp(4 if compact else 6)
        full_left = self.x + margin
        full_bottom = self.y + margin
        full_w = max(dp(180), self.width - margin * 2)
        full_h = max(dp(150), self.height - margin * 2)

        w, h = dp(92), dp(22)
        x = self.center_x - w / 2
        y = self.center_y - h / 2
        self.placeholder_label.pos = (x, y)
        self._chip.pos = (x - dp(10), y - dp(6))
        self._chip.size = (w + dp(20), h + dp(12))

        overlay_lane_h = WORK_OVERLAY_LANE_H if compact else dp(8)
        panel_h = max(dp(120), full_h - overlay_lane_h)
        self.map_panel.size = (full_w, panel_h)
        self.map_panel.pos = (full_left, full_bottom)
        self.map_panel.set_compact_mode(compact or panel_h <= dp(520))


class MainRailPanel(BoxLayout):
    def __init__(self, on_submenu_action=None, **kwargs):
        kwargs.setdefault('orientation', 'vertical')
        kwargs.setdefault('size_hint', (None, 1))
        kwargs.setdefault('width', RAIL_W + dp(6))
        kwargs.setdefault('spacing', RAIL_GAP)
        kwargs.setdefault('padding', [0, RAIL_TOP_PAD, 0, dp(8)])
        super().__init__(**kwargs)
        self.rows = []
        self.on_submenu_action = on_submenu_action

        with self.canvas.before:
            Color(*BG_RAIL)
            self._rail_bg = Rectangle(pos=self.pos, size=(self.width, self.height))
        self.bind(pos=self._sync_bg, size=self._sync_bg)

        self.title_label = Label(
            text='MAIN',
            size_hint=(None, None),
            size=(self.width, TITLE_H),
            color=LINE_ACCENT,
            bold=False,
            font_size='10sp',
            halign='center',
            valign='middle',
        )
        vazir_font = ui_font(FONT_VAZIR)
        if vazir_font:
            self.title_label.font_name = vazir_font
        self.title_label.bind(size=lambda w, s: setattr(w, 'text_size', s))
        self.add_widget(self.title_label)

        row_specs = [
            ('', 'satellite'),
            ('02', None),
            ('03', None),
            ('04', None),
            ('05', None),
            ('06', None),
            ('07', None),
            ('08', None),
        ]
        for idx, (caption, icon_kind) in enumerate(row_specs, start=1):
            row = MainTouchRow(
                caption=caption,
                main_index=idx,
                on_toggle=self._handle_row_toggle,
                on_submenu_pick=self._handle_submenu_pick,
                icon_kind=icon_kind,
            )
            self.rows.append(row)
            self.add_widget(row)

        self.add_widget(Widget())

    def _handle_row_toggle(self, target_row, want_open):
        if want_open:
            for row in self.rows:
                row._tap_scheduled = False
                row.set_open(row is target_row)
        else:
            target_row._tap_scheduled = False
            target_row.set_open(False)

    def _handle_submenu_pick(self, main_index, child_index, spec):
        if self.on_submenu_action:
            self.on_submenu_action(main_index, child_index, spec)

    def _sync_bg(self, *_):
        self._rail_bg.pos = self.pos
        self._rail_bg.size = (self.width, self.height)
        self.title_label.width = self.width


class ShellBody(FloatLayout):
    def __init__(self, status_callback=None, **kwargs):
        super().__init__(**kwargs)
        self.status_callback = status_callback
        self._gps_busy = False
        self._gps_request_id = 0
        self._gps_request_started_at = 0.0
        self._gps_watchdog = None
        self._gps_poll_event = None
        self._gps_live_session = None
        self._gps_probe = {}
        self._gps_detail_lines = []
        self._gps_best_fix = None
        self._last_gps_fix = None
        self.native_map = AndroidWebMapOverlay(status_callback=self._post_status) if runtime_map_mode() == 'android_inapp' else NullMapRuntime()

        with self.canvas.before:
            Color(0.08, 0.10, 0.13, 1)
            self._body_bg = Rectangle(pos=self.pos, size=self.size)
        self.bind(pos=self._layout_body, size=self._layout_body)

        self.workspace = WorkspaceHost(size_hint=(None, None))
        self.add_widget(self.workspace)
        try:
            self.workspace.map_panel.view.bind(pos=self._sync_native_map_viewport, size=self._sync_native_map_viewport)
        except Exception:
            pass

        self.main_panel = MainRailPanel(on_submenu_action=self._handle_main_submenu_action, size_hint=(None, None))
        self.add_widget(self.main_panel)
        self._layout_body()

    def _post_status(self, text):
        self.workspace.set_map_status(text)
        if self.status_callback:
            self.status_callback(text)

    def _layout_body(self, *_):
        self._body_bg.pos = self.pos
        self._body_bg.size = self.size
        pad = dp(4)
        gap = dp(2)
        body_h = max(dp(180), self.height - pad * 2)
        rail_shell_w = max(dp(80), min(dp(88), self.width * 0.145))
        self.main_panel.pos = (self.x + pad, self.y + pad)
        self.main_panel.size = (rail_shell_w, body_h)
        work_x = self.x + pad + rail_shell_w + gap
        work_w = max(dp(180), self.right - work_x - pad)
        self.workspace.pos = (work_x, self.y + pad)
        self.workspace.size = (work_w, body_h)
        self._sync_native_map_viewport()

    def _sync_native_map_viewport(self, *_):
        try:
            self.native_map.update_viewport(self._native_map_viewport())
        except Exception:
            pass

    def _native_map_viewport(self):
        target = getattr(getattr(self.workspace, 'map_panel', None), 'view', None) or self.workspace
        try:
            kivy_w = float(Window.width)
            kivy_h = float(Window.height)
        except Exception:
            kivy_w = float(max(1.0, self.width))
            kivy_h = float(max(1.0, self.height))

        scale_x = 1.0
        scale_y = 1.0
        try:
            activity, _context, _activity_name = get_android_activity_context()
            if activity is not None:
                display = activity.getResources().getDisplayMetrics()
                native_w = float(getattr(display, 'widthPixels', 0) or 0)
                native_h = float(getattr(display, 'heightPixels', 0) or 0)
                if native_w > 0 and kivy_w > 0:
                    scale_x = native_w / kivy_w
                if native_h > 0 and kivy_h > 0:
                    scale_y = native_h / kivy_h
        except Exception:
            pass

        left = int(round(float(target.x) * scale_x))
        right = int(round(max(0.0, kivy_w - float(target.right)) * scale_x))
        bottom = int(round(float(target.y) * scale_y))
        top = int(round(max(0.0, kivy_h - float(target.top)) * scale_y))
        return {'left': max(0, left), 'top': max(0, top), 'right': max(0, right), 'bottom': max(0, bottom)}

    def _map_focus_latlon(self):
        if self._last_gps_fix is not None:
            return float(self._last_gps_fix['lat']), float(self._last_gps_fix['lon'])
        if self.workspace.current_lat is not None and self.workspace.current_lon is not None:
            return float(self.workspace.current_lat), float(self.workspace.current_lon)
        return DEFAULT_MAP_LAT, DEFAULT_MAP_LON

    def _open_native_provider_map(self, style_key):
        target = map_runtime_target(style_key)
        lat, lon = self._map_focus_latlon()
        provider = self._last_gps_fix['provider'] if self._last_gps_fix else ''
        self._cancel_pending_gps_request(f'open_{style_key}_map')
        line1, line2, line3, note_text, status_text = runtime_map_message(target)
        self.workspace.show_map_panel('MY LOCATION', target['style_label'])
        profile = current_runtime_profile()
        target_zoom = SAT_INTERNAL_DEFAULT_ZOOM if target['style_key'] == 'satellite' else GPS_PARCEL_ZOOM
        self.workspace.set_map_style(target['ui_style_key'], target['style_label'])
        self.workspace.set_map_zoom(target_zoom)
        self.workspace.set_map_coordinates(lat, lon, provider, reset_zoom=False, target_zoom=target_zoom)
        viewport = self._native_map_viewport()
        mode = runtime_map_mode()

        if profile.get('is_pydroid_like'):
            try:
                self.native_map.hide()
            except Exception:
                pass
            self.workspace.set_map_probe(f"mode=pydroid_kivy_fallback | style={target['ui_style_key']}\n" + runtime_profile_probe_text(profile))
            self.workspace.set_map_note('Pydroid fallback branch: the internal PyStation-style Kivy map is active inside the app panel. Satellite now keeps a closer view by overzooming stable source tiles instead of insisting on unstable direct level-20 imagery.')
            self._post_status(f"GPS > {target['style_key'].upper()} | internal Kivy map active | clipped to workspace | stable overzoom active")
            return

        if mode == 'android_inapp':
            payload = build_in_app_map_payload(lat, lon, target_zoom, target['ui_style_key'])
            if target['style_key'] == 'satellite':
                if payload.get('provider') == 'iran_custom_satellite':
                    note_text = 'Android in-app satellite overlay is opening inside the app frame with the configured Iranian/custom satellite tile source.'
                else:
                    note_text = 'Android in-app satellite overlay is opening inside the app frame. Until IRAN_SATELLITE_TILE_URL is inserted, the in-app satellite path uses a public imagery fallback.'
            else:
                if payload.get('provider') == 'neshan_road':
                    note_text = 'Android in-app road overlay is opening inside the app frame through the Neshan web SDK.'
                else:
                    note_text = 'Android in-app road overlay is opening inside the app frame with a public road fallback until NESHAN_WEB_API_KEY is inserted.'
            self.workspace.set_map_shell_message(f"{line1}\n{line2}\nOPENING: {payload.get('label', 'MAP')}")
            self.workspace.set_map_note(note_text)
            self.workspace.set_map_probe(f"mode=android_inapp | payload={payload.get('provider', 'unknown')}\n" + runtime_profile_probe_text(profile))
            self._post_status(f"{status_text} | opening {payload.get('label', 'MAP')}...")
            opened = self.native_map.show(lat, lon, zoom=target_zoom, style_key=target['ui_style_key'], viewport=viewport)
            if not opened:
                self.workspace.set_map_shell_message(f"{line1}\n{line2}\nNATIVE MAP OPEN FAILED")
                self.workspace.set_map_note('The Android runtime did not expose the in-app WebView overlay. If this happens on-device, inspect inapp_webview_error.txt after the run.')
                self._post_status(f"{status_text} | native overlay unavailable")
            return

        self.workspace.set_map_shell_message(f"{line1}\n{line2}\n{line3}")
        self.workspace.set_map_note(note_text)
        self.workspace.set_map_probe(runtime_profile_probe_text(profile))
        self._post_status(status_text)

    def _close_native_provider_map(self):
        try:
            self.native_map.hide()
        except Exception:
            pass

    def _clear_gps_timers(self):
        if self._gps_poll_event is not None:
            try:
                self._gps_poll_event.cancel()
            except Exception:
                pass
            self._gps_poll_event = None
        if self._gps_watchdog is not None:
            try:
                self._gps_watchdog.cancel()
            except Exception:
                pass
            self._gps_watchdog = None

    def _close_gps_live_session(self):
        if self._gps_live_session is not None:
            try:
                close_pyjnius_live_session(self._gps_live_session)
            except Exception:
                pass
            self._gps_live_session = None

    def _cancel_pending_gps_request(self, reason=''):
        had_pending = bool(
            self._gps_busy or
            self._gps_poll_event is not None or
            self._gps_watchdog is not None or
            self._gps_live_session is not None
        )
        if had_pending:
            self._gps_request_id += 1
        self._gps_busy = False
        self._clear_gps_timers()
        self._close_gps_live_session()
        if had_pending and reason:
            try:
                self._post_status(f'Map launch is taking priority over GPS refresh | {reason}')
            except Exception:
                pass

    def _result_from_fix(self, fix, backend='pyjnius', message='GPS fix received', weak_fix=False):
        return {
            'ok': True,
            'lat': fix[0],
            'lon': fix[1],
            'accuracy': fix[2],
            'provider': fix[3],
            'backend': backend,
            'probe': dict(self._gps_probe),
            'detail': '\n'.join(self._gps_detail_lines),
            'message': message,
            'weak_fix': weak_fix,
        }

    def _fail_result(self, message):
        return {
            'ok': False,
            'message': message,
            'probe': dict(self._gps_probe),
            'detail': '\n'.join(self._gps_detail_lines),
        }

    def _finish_gps_request(self, request_id, result):
        if request_id != self._gps_request_id:
            return
        self._clear_gps_timers()
        self._close_gps_live_session()
        self._gps_busy = False
        self._apply_gps_result(result)

    def _handle_main_submenu_action(self, main_index, child_index, spec):
        try:
            if main_index != 1:
                return

            action = spec.get('action')
            if action == 'my_location':
                self._close_native_provider_map()
                self.workspace.show_map_panel('MY LOCATION', 'SATELLITE')
                self.workspace.set_map_style('satellite', 'SATELLITE')
                self.workspace.set_map_zoom(GPS_PARCEL_ZOOM)
                if self._last_gps_fix:
                    self.workspace.set_map_coordinates(self._last_gps_fix['lat'], self._last_gps_fix['lon'], self._last_gps_fix['provider'], reset_zoom=True, target_zoom=GPS_PARCEL_ZOOM)
                    self.workspace.set_map_note('Stored GPS fix is visible in the shell panel. A fresh GPS request is starting.')
                    self._post_status('GPS > MY LOCATION | refreshing GPS into shell panel...')
                else:
                    self.workspace.set_map_coordinates(None, None, '')
                    self.workspace.set_map_note('Using the stable GPS backend first. The panel stays visible while the GPS request starts.')
                    self._post_status('GPS > MY LOCATION | preparing GPS request...')
                self.workspace.set_map_probe('permission request will start here...')
                self._request_live_location()
            elif action == 'satellite':
                self._open_native_provider_map('satellite')
            elif action == 'road':
                self._open_native_provider_map('road')
            elif action == 'boundary':
                self._close_native_provider_map()
                self.workspace.show_map_panel('GPS PANEL', 'APK BUILD NOTES')
                self.workspace.set_map_style('bound', 'APK BUILD NOTES')
                self.workspace.set_map_shell_message('APK BUILD NOTES\nCurrent runtime cannot host in-app satellite WebView')
                self.workspace.set_map_note('APK branch note: keep this runtime only for shell and GPS checks. In-app map activation belongs to the packaged build.')
                self._post_status('GPS > BOUND | APK build notes')
            elif action == 'pin':
                self._close_native_provider_map()
                self.workspace.show_map_panel('GPS PANEL', 'APK BUILD NOTES')
                self.workspace.set_map_style('hybrid', 'APK BUILD NOTES')
                self.workspace.set_map_shell_message('APK BUILD NOTES\nSatellite first, ROAD second')
                self.workspace.set_map_note('Provider note: ROAD remains Neshan-first. SAT remains planned for the packaged APK runtime path.')
                self._post_status('GPS > PIN | APK build notes')
        except Exception as exc:
            self.workspace.show_map_panel('GPS PANEL', 'SAFE MODE')
            self.workspace.set_map_style('road', 'SAFE MODE')
            self.workspace.set_map_shell_message(f'SAFE MODE ERROR\n{exc.__class__.__name__}: {exc}')
            self.workspace.set_map_note('A submenu action was trapped safely so the app should stay open.')
            self._post_status(f'SAFE MODE | submenu action trapped | {exc.__class__.__name__}: {exc}')

    def _request_live_location(self):
        if self._gps_busy:
            elapsed = max(0.0, time.time() - self._gps_request_started_at)
            if self._last_gps_fix is not None:
                self.workspace.set_map_coordinates(self._last_gps_fix['lat'], self._last_gps_fix['lon'], self._last_gps_fix['provider'], reset_zoom=False, target_zoom=GPS_PARCEL_ZOOM)
            if elapsed < 18.0:
                self._post_status(f'GPS refresh already running... {elapsed:.0f}s')
                return
            self._post_status('Previous GPS request timed out. Restarting...')
            self._clear_gps_timers()
            self._close_gps_live_session()
            self._gps_busy = False

        self._gps_busy = True
        self._gps_request_id += 1
        request_id = self._gps_request_id
        self._gps_request_started_at = time.time()
        self._gps_probe = {}
        self._gps_detail_lines = []
        self._gps_best_fix = None
        self._last_gps_fix = None
        self._clear_gps_timers()
        self._close_gps_live_session()

        requested = False
        try:
            requested = request_device_location_permissions()
        except Exception as exc:
            self._gps_detail_lines.append(f'permission_request_error: {exc.__class__.__name__}')
        if requested:
            self.workspace.set_map_probe('permission requested\nstarting GPS...')
            self._post_status('GPS > MY LOCATION | Permission requested, waiting to start GPS...')
        else:
            self.workspace.set_map_probe('permission helper unavailable\nstarting GPS directly...')
            self._post_status('GPS > MY LOCATION | Starting GPS directly...')

        self._gps_watchdog = Clock.schedule_once(lambda dt, req=request_id: self._gps_request_watchdog(req), 18.0)
        Clock.schedule_once(lambda dt, req=request_id: self._begin_live_location_after_permission(req), 1.1)

    def _begin_live_location_after_permission(self, request_id):
        if request_id != self._gps_request_id or not self._gps_busy:
            return

        try:
            self._gps_probe = probe_runtime_modules()
            probe_multiline = self._gps_probe.get('multiline', '')
            if probe_multiline:
                self.workspace.set_map_probe(probe_multiline)

            last_known_fix, last_known_report = try_get_pyjnius_last_known_fix()
            if last_known_report:
                self._gps_detail_lines.append(f'pyjnius_last_known: {last_known_report}')
            self._gps_best_fix = better_location_fix(self._gps_best_fix, last_known_fix)

            if is_good_device_fix(self._gps_best_fix):
                result = self._result_from_fix(self._gps_best_fix, backend='pyjnius', message='Good last-known fix received', weak_fix=False)
                self._finish_gps_request(request_id, result)
                return

            if self._gps_best_fix is not None:
                self.workspace.set_map_coordinates(self._gps_best_fix[0], self._gps_best_fix[1], self._gps_best_fix[3])
                self._post_status(
                    f'LAST KNOWN FIX | lat={self._gps_best_fix[0]:.6f} lon={self._gps_best_fix[1]:.6f} '
                    f'provider={self._gps_best_fix[3]} acc={self._gps_best_fix[2]:.1f}m | waiting for live GPS...'
                )
            else:
                self._post_status('No last-known fix yet | starting live PyJNIus GPS...')

            session, live_report, live_error = begin_pyjnius_live_session(timeout_sec=10)
            if live_report:
                self._gps_detail_lines.append(f'pyjnius_live: {live_report}')
            if session is None:
                if live_error:
                    self._gps_detail_lines.append(f'pyjnius_live_error: {live_error}')
                self._start_bridge_fallback(request_id)
                return

            self._gps_live_session = session
            self._gps_poll_event = Clock.schedule_interval(lambda dt, req=request_id: self._poll_live_location(req, dt), 0.45)
            self._post_status('PyJNIus live GPS started...')
        except Exception as exc:
            self._gps_detail_lines.append(f'gps_begin_error: {exc.__class__.__name__}: {exc}')
            self._finish_gps_request(request_id, self._fail_result(f'GPS begin error: {exc}'))

    def _poll_live_location(self, request_id, _dt):
        if request_id != self._gps_request_id or not self._gps_busy:
            return False
        session = self._gps_live_session
        if session is None:
            return False

        try:
            live_fix = poll_pyjnius_live_session(session)
        except Exception as exc:
            self._gps_detail_lines.append(f'pyjnius_poll_error: {exc.__class__.__name__}: {exc}')
            self._close_gps_live_session()
            self._start_bridge_fallback(request_id)
            return False

        self._gps_best_fix = better_location_fix(self._gps_best_fix, live_fix)
        if is_good_device_fix(self._gps_best_fix):
            result = self._result_from_fix(self._gps_best_fix, backend='pyjnius', message='Live GPS fix received via PyJNIus', weak_fix=False)
            self._finish_gps_request(request_id, result)
            return False

        if self._gps_best_fix is not None:
            self.workspace.set_map_coordinates(self._gps_best_fix[0], self._gps_best_fix[1], self._gps_best_fix[3])

        if time.time() >= session.get('deadline', 0):
            self._close_gps_live_session()
            self._start_bridge_fallback(request_id)
            return False
        return True

    def _start_bridge_fallback(self, request_id):
        if request_id != self._gps_request_id or not self._gps_busy:
            return

        if not self._gps_probe.get('bridge'):
            if self._gps_best_fix is not None:
                result = self._result_from_fix(self._gps_best_fix, backend='pyjnius', message='Weak last-known fix only', weak_fix=True)
            else:
                result = self._fail_result('No device location fix available - check GPS permission and phone location settings')
            self._finish_gps_request(request_id, result)
            return

        self._post_status('PyJNIus did not finish with a strong fix | trying Android bridge fallback...')
        details_snapshot = list(self._gps_detail_lines)
        best_snapshot = self._gps_best_fix
        probe_snapshot = dict(self._gps_probe)
        worker = threading.Thread(
            target=self._gps_bridge_worker,
            args=(request_id, details_snapshot, best_snapshot, probe_snapshot),
            daemon=True,
        )
        worker.start()

    def _gps_bridge_worker(self, request_id, detail_lines, best_fix, probe_snapshot):
        try:
            bridge_fix, bridge_report, bridge_module = try_get_android_device_location(timeout_sec=6)
            if bridge_report:
                detail_lines.append(f'android_bridge: {bridge_report}')
            merged_fix = better_location_fix(best_fix, bridge_fix)
            if merged_fix is not None:
                if best_fix is not None and (bridge_fix is None or location_fix_rank(best_fix) <= location_fix_rank(bridge_fix)):
                    backend = 'pyjnius'
                else:
                    backend = bridge_module or 'android'
                result = {
                    'ok': True,
                    'lat': merged_fix[0],
                    'lon': merged_fix[1],
                    'accuracy': merged_fix[2],
                    'provider': merged_fix[3],
                    'backend': backend,
                    'probe': probe_snapshot,
                    'detail': '\n'.join(detail_lines),
                    'message': 'GPS fix received after fallback',
                    'weak_fix': not is_good_device_fix(merged_fix),
                }
            else:
                result = {
                    'ok': False,
                    'message': 'No device location fix available - check GPS permission and phone location settings',
                    'probe': probe_snapshot,
                    'detail': '\n'.join(detail_lines),
                }
        except Exception as exc:
            result = {
                'ok': False,
                'message': f'GPS fallback error: {exc}',
                'probe': probe_snapshot,
                'detail': '\n'.join(detail_lines + [f'android_bridge_error: {exc.__class__.__name__}: {exc}']),
            }
        Clock.schedule_once(lambda dt, req=request_id, payload=result: self._finish_gps_request(req, payload), 0)

    def _gps_request_watchdog(self, request_id):
        if request_id != self._gps_request_id or not self._gps_busy:
            return
        self._gps_detail_lines.append('watchdog: request timeout')
        if self._gps_best_fix is not None:
            result = self._result_from_fix(self._gps_best_fix, backend='pyjnius', message='GPS request timeout - returning best fix so far', weak_fix=True)
        else:
            result = self._fail_result('GPS request timeout - no fix received')
        self._finish_gps_request(request_id, result)

    def _apply_gps_result(self, result):
        probe = result.get('probe', {})
        probe_summary = probe.get('summary', '')
        probe_multiline = probe.get('multiline', '')
        detail = result.get('detail', '')
        if probe_multiline:
            self.workspace.set_map_probe(probe_multiline)
        if detail:
            self.workspace.set_map_note(detail)

        self.workspace.show_map_panel('MY LOCATION', 'GPS SHELL')
        self.workspace.set_map_style('satellite', 'GPS SHELL')
        self.workspace.set_map_zoom(GPS_PARCEL_ZOOM)
        if result.get('ok'):
            lat = result.get('lat')
            lon = result.get('lon')
            provider = result.get('provider', '')
            backend = result.get('backend', '')
            acc = result.get('accuracy', 0.0)
            self._last_gps_fix = {'lat': lat, 'lon': lon, 'provider': provider, 'accuracy': acc, 'backend': backend}
            self.workspace.set_map_coordinates(lat, lon, provider, reset_zoom=True, target_zoom=GPS_PARCEL_ZOOM)
            if getattr(self.native_map, 'is_visible', False):
                try:
                    self.native_map.show(lat, lon, zoom=self.workspace.current_zoom, style_key=getattr(self.native_map, 'current_style_key', 'satellite'), viewport=self._native_map_viewport())
                except Exception:
                    pass
            self.workspace.set_map_shell_message('GPS / UTM MAP\nTap SAT or ROAD to load the internal in-app map')
            self.workspace.set_map_note('GPS fix stored. Tap SAT for satellite or ROAD for topo inside the same app panel. Internal Kivy map fallback is active for Pydroid tests.')
            quality = 'WEAK FIX' if result.get('weak_fix') else 'LIVE FIX'
            self._post_status(f"{quality} | lat={lat:.6f} lon={lon:.6f} provider={provider} acc={acc:.1f}m backend={backend}")
        else:
            self.workspace.set_map_coordinates(None, None, '')
            if probe_summary:
                self._post_status(f"{result.get('message', 'GPS unavailable')} | {probe_summary}")
            else:
                self._post_status(result.get('message', 'GPS unavailable'))


class RootShell(FloatLayout):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)

        with self.canvas.before:
            Color(*BG_APP)
            self._bg = Rectangle(pos=self.pos, size=self.size)
        self.bind(pos=self._layout_shell, size=self._layout_shell)

        self.top_bar = TopBar(status_callback=self.set_status, size_hint=(None, None))
        self.add_widget(self.top_bar)

        self.body = ShellBody(status_callback=self.set_status, size_hint=(None, None))
        self.add_widget(self.body)

        self.status_bar = StatusBar(size_hint=(None, None))
        self.add_widget(self.status_bar)
        self._layout_shell()
        self.set_status('READY')

    def set_status(self, text):
        self.status_bar.set_message(text)

    def _layout_shell(self, *_):
        self._bg.pos = self.pos
        self._bg.size = self.size
        top_h = TOP_H
        status_h = STATUS_H
        self.top_bar.pos = (self.x, self.top - top_h)
        self.top_bar.size = (self.width, top_h)
        self.status_bar.pos = (self.x, self.y)
        self.status_bar.size = (self.width, status_h)
        body_y = self.y + status_h
        body_h = max(dp(160), self.height - top_h - status_h)
        self.body.pos = (self.x, body_y)
        self.body.size = (self.width, body_h)


class RewriteShellApp(App):
    def build(self):
        self.title = 'r92 - PyStation Stable Satellite Overzoom Fix'
        self.root_shell = RootShell()
        return self.root_shell

    def on_stop(self):
        try:
            self.root_shell.body.native_map.dispose()
        except Exception:
            pass


if __name__ == '__main__':
    print('=' * 56)
    print('r92 - PyStation Stable Satellite Overzoom Fix')
    print('Scope: keep the clipped internal Kivy tile map fallback, but stabilize SAT at close zoom by reusing stable lower-level source tiles instead of requesting missing direct imagery tiles')
    print('FILE: NEW | OPEN | SAVE | SAVE AS | IMPORT | EXPORT | EXIT')
    print('=' * 56)
    RewriteShellApp().run()
