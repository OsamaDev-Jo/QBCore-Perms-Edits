# QBCore-Perms-Edits
This edit make the perm saves on data and optional based on citizen id or licence 

```markdown
# 🛠️ QB-Core Advanced Permission System

![QB-Core](https://img.shields.io/badge/QBCore-Compatible-blue)
![License](https://img.shields.io/badge/License-GPL--3.0-orange)

## 📌 Features
- ✅ Persistent permission storage
- 🔄 Automatic permission synchronization
- 👮‍♂️ CitizenID **or** License based (configurable)
- 📊 MySQL database integration
- 🔒 ACE permission system support

## ⚙️ Installation

### 1️⃣ Modify QB-Core Functions
Replace these in `qb-core/server/functions.lua`:

```lua
-- CONFIG: Set to true to use License instead of CitizenID
local USE_LICENSE_ID = false

function QBCore.Functions.AddPermission(source, permission)
    local src = source
    local Player = QBCore.Functions.GetPlayer(src)
    if not Player then return end
    
    local identifier = USE_LICENSE_ID and GetPlayerIdentifier(src, 'license') or Player.PlayerData.citizenid
    local identifierType = USE_LICENSE_ID and 'license' or 'citizenid'
    
    QBCore.Config.Server.Permissions[identifier] = {
        identifier = identifier,
        permission = permission:lower()
    }

    MySQL.Async.execute('DELETE FROM permissions WHERE identifier = ?', { identifier })
    MySQL.Async.insert('INSERT INTO permissions (name, identifier, permission, type) VALUES (?, ?, ?, ?)', {
        GetPlayerName(src),
        identifier,
        permission:lower(),
        identifierType
    })
    
    ExecuteCommand(('add_principal identifier.%s qbcore.%s'):format(identifier, permission))
    QBCore.Commands.Refresh(src)
    TriggerClientEvent('QBCore:Client:OnPermissionUpdate', src, permission)
end

function QBCore.Functions.RemovePermission(source, permission)
    local src = source
    local Player = QBCore.Functions.GetPlayer(src)
    if not Player then return end
    
    local identifier = USE_LICENSE_ID and GetPlayerIdentifier(src, 'license') or Player.PlayerData.citizenid
    
    if permission then
        if IsPlayerAceAllowed(src, permission) then
            ExecuteCommand(('remove_principal identifier.%s qbcore.%s'):format(identifier, permission))
            MySQL.Async.execute('DELETE FROM permissions WHERE identifier = ?', { identifier })
            QBCore.Commands.Refresh(src)
        end
    else
        for _, v in pairs(QBCore.Config.Server.Permissions) do
            if IsPlayerAceAllowed(src, v) then
                ExecuteCommand(('remove_principal identifier.%s qbcore.%s'):format(identifier, v))
                MySQL.Async.execute('DELETE FROM permissions WHERE identifier = ?', { identifier })
                QBCore.Commands.Refresh(src)
            end
        end
    end
end
```

### 2️⃣ Database Setup
```sql
CREATE TABLE IF NOT EXISTS `permissions` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `identifier` VARCHAR(255) NOT NULL,
  `name` VARCHAR(50) DEFAULT NULL,
  `permission` VARCHAR(50) DEFAULT 'user',
  `type` ENUM('citizenid','license') DEFAULT 'citizenid',
  PRIMARY KEY (`id`),
  UNIQUE KEY `identifier` (`identifier`),
  KEY `permission` (`permission`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 🔄 Permission Restoration
Add this to your server startup:

```lua
MySQL.ready(function()
    local permissions = MySQL.Sync.fetchAll('SELECT * FROM permissions')
    for _, data in ipairs(permissions) do
        QBCore.Config.Server.Permissions[data.identifier] = {
            identifier = data.identifier,
            permission = data.permission
        }
        ExecuteCommand(('add_principal identifier.%s qbcore.%s'):format(data.identifier, data.permission))
    end
end)
```

## 📝 License
This project is licensed under GPL-3.0. Please include attribution if modifying/distributing.

## ⚠️ Important Notes
1. Set `USE_LICENSE_ID` to true if you prefer license-based permissions
```
