UHDadmin/
├─ pyproject.toml
├─ requirements.txt
├─ config.toml
├─ main.py
├─ jwt_blacklist.sql                # (如果想单独保存建表或清空脚本)
└─ app/
   ├─ __init__.py
   ├─ config.py
   ├─ utils.py
   ├─ main.py                       # 也可把main放此处, 视项目需要
   ├─ services/
   │  ├─ __init__.py
   │  └─ auth.py
   ├─ models/
   │  ├─ __init__.py
   │  ├─ users.py
   │  ├─ roles.py
   │  ├─ user_roles.py
   │  ├─ datetime_model.py
   │  ├─ invitation.py
   │  ├─ emby_account.py
   │  ├─ log_emby_account.py
   │  └─ system_logs.py
   ├─ schemas/
   │  ├─ __init__.py
   │  ├─ auth.py
   │  ├─ users.py
   │  ├─ roles.py
   │  ├─ user_roles.py
   │  ├─ datetime_schemas.py
   │  ├─ invitations.py
   │  ├─ emby_account.py
   │  ├─ system_logs.py
   │  └─ statistics.py
   └─ routers/
      ├─ __init__.py
      ├─ auth.py
      ├─ user.py
      └─ admin/
         ├─ __init__.py
         ├─ roles.py
         ├─ users.py
         ├─ logs.py
         ├─ activities.py
         └─ statistics.py
