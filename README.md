# Test_task
Веб приложение для управления системными сервисами

Ниже представлен код на Python для работы со службой Docker на основе FastAPI.
Программа предоставляет API для получения текущего статуса службы, а также её запуска, остановки и перезапуска.
Реализован контроль доступа: операции с сервисом возможны только при включённом разрешении, статус которого сохраняется между перезапусками приложения.

from fastapi import FastAPI, HTTPException
import subprocess
import json
import os

app = FastAPI(title="Service Manager", description="Управление службой Docker через API")

SERVICE_NAME = "docker"
ACCESS_FILE = "access.json"

def load_access():
    if os.path.exists(ACCESS_FILE):
        with open(ACCESS_FILE, "r") as f:
            return json.load(f).get("enabled", False)
    return False

def save_access(enabled: bool):
    with open(ACCESS_FILE, "w") as f:
        json.dump({"enabled": enabled}, f)

def require_access():
    if not load_access():
        raise HTTPException(status_code=403, detail="Доступ запрещён")

def run_systemctl(command: str):
    result = subprocess.run(
        ["systemctl", command, SERVICE_NAME],
        capture_output=True, text=True
    )
    return result

def get_status():
    result = subprocess.run(
        ["systemctl", "is-active", SERVICE_NAME],
        capture_output=True, text=True
    )
    return result.stdout.strip()

@app.get("/status")
def status():
    """Получить статус Docker"""
    return {"service": SERVICE_NAME, "status": get_status()}


@app.post("/start")
def start_service():
    require_access()
    run_systemctl("start")
    return {"status": get_status()}


@app.post("/stop")
def stop_service():
    require_access()
    run_systemctl("stop")
    return {"status": get_status()}


@app.post("/restart")
def restart_service():
    require_access()
    run_systemctl("restart")
    return {"status": get_status()}


@app.get("/access")
def access_status():
    """Проверить доступ"""
    return {"enabled": load_access()}


@app.post("/access/enable")
def enable_access():
    save_access(True)
    return {"enabled": True}


@app.post("/access/disable")
def disable_access():
    save_access(False)
    return {"enabled": False}
