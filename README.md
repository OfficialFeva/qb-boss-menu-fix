# qb-boss-menu-fix
Quick fix that bridges the gap between qb-banking and qb-management in order to display business/gang bank accounts in the boss menu.

# Feva's QB-Core Boss Menu Financial Integration Guide 
# Show some love <3

This guide explains how to implement financial information integration into the QB-Core boss menu system, allowing business owners and gang leaders to view their account balances directly from the management interface.

## Overview

In recent QB-Core updates, financial management was migrated from `qb-management` to `qb-banking`. This change created a disconnect between the boss menu and financial information. This guide provides a solution that bridges this gap by displaying account balances in the boss menu.

## Prerequisites

- A working QB-Core server
- Access to server files
- Basic understanding of Lua
- Required resources:
  - qb-management
  - qb-banking
  - qb-core

## Implementation Steps

### Step 1: Add Balance Callbacks to Banking System

First, we need to create server-side callbacks in the banking system to securely fetch account balances.

Open `qb-banking/server.lua` and add the following code at the end of the file:

```lua
-- Callback to get the job account balance for boss menu
QBCore.Functions.CreateCallback('qb-banking:server:getBossAccountBalance', function(source, cb, accountName)
    local src = source
    local Player = QBCore.Functions.GetPlayer(src)
    
    if not Player then return cb(0) end
    if not Player.PlayerData.job.isboss then return cb(0) end
    if Player.PlayerData.job.name ~= accountName then return cb(0) end
    
    local account = GetAccount(accountName)
    if not account then return cb(0) end
    
    cb(account.account_balance)
end)

-- Callback to get the gang account balance for gang menu
QBCore.Functions.CreateCallback('qb-banking:server:getGangAccountBalance', function(source, cb, accountName)
    local src = source
    local Player = QBCore.Functions.GetPlayer(src)
    
    if not Player then return cb(0) end
    if not Player.PlayerData.gang.isboss then return cb(0) end
    if Player.PlayerData.gang.name ~= accountName then return cb(0) end
    
    local account = GetAccount(accountName)
    if not account then return cb(0) end
    
    cb(account.account_balance)
end)
```

These callbacks retrieve the account balance and include security checks to ensure that only authorized users can access the information.

### Step 2: Update the Boss Menu

Next, we'll modify the boss menu to display the account balance.

Open `qb-management/client/cl_boss.lua` and find the `RegisterNetEvent('qb-bossmenu:client:OpenMenu', function()` event. Replace it with the following code:

```lua

RegisterNetEvent('qb-bossmenu:client:OpenMenu', function()
    if not PlayerJob.name or not PlayerJob.isboss then return end

    -- Get account balance from banking system
    QBCore.Functions.TriggerCallback('qb-banking:server:getBossAccountBalance', function(balance)
        local bossMenu = {
            {
                header = Lang:t('headers.bsm') .. string.upper(PlayerJob.label),
                icon = 'fa-solid fa-circle-info',
                isMenuHeader = true,
            },
            {
                header = PlayerJob.label .. " Funds: $" .. format_int(balance),
                txt = "",
                icon = 'fa-solid fa-money-bill-wave',
                isMenuHeader = true,
            },
            {
                header = Lang:t('body.manage'),
                txt = Lang:t('body.managed'),
                icon = 'fa-solid fa-list',
                params = {
                    event = 'qb-bossmenu:client:employeelist',
                }
            },
            {
                header = Lang:t('body.hire'),
                txt = Lang:t('body.hired'),
                icon = 'fa-solid fa-hand-holding',
                params = {
                    event = 'qb-bossmenu:client:HireMenu',
                }
            },
            {
                header = Lang:t('body.storage'),
                txt = Lang:t('body.storaged'),
                icon = 'fa-solid fa-box-open',
                params = {
                    isServer = true,
                    event = 'qb-bossmenu:server:stash',
                }
            },
            {
                header = Lang:t('body.outfits'),
                txt = Lang:t('body.outfitsd'),
                icon = 'fa-solid fa-shirt',
                params = {
                    event = 'qb-bossmenu:client:Wardrobe',
                }
            }
        }

        for _, v in pairs(DynamicMenuItems) do
            bossMenu[#bossMenu + 1] = v
        end

        bossMenu[#bossMenu + 1] = {
            header = Lang:t('body.exit'),
            icon = 'fa-solid fa-angle-left',
            params = {
                event = 'qb-menu:closeMenu',
            }
        }

        exports['qb-menu']:openMenu(bossMenu)
    end, PlayerJob.name)
end)

-- Helper function to format integers
function format_int(number)
    local i, j, minus, int, fraction = tostring(number):find('([-]?)(%d+)([.]?%d*)')
    int = int:reverse():gsub("(%d%d%d)", "%1,")
    return minus .. int:reverse():gsub("^,", "") .. fraction
end
```

This code adds a new menu item showing the company's account balance. The `format_int` function ensures the balance is displayed with commas for better readability.

### Step 3: Update the Gang Menu

Similar to the boss menu, we'll update the gang menu to display the gang's account balance.

Open `qb-management/client/cl_gang.lua` and find the `RegisterNetEvent('qb-gangmenu:client:OpenMenu', function()` event. Replace it with the following code:

```lua
-- Helper function to format integers with commas for thousands (if not already defined in this file)
function format_int(number)
    local i, j, minus, int, fraction = tostring(number):find('([-]?)(%d+)([.]?%d*)')
    int = int:reverse():gsub("(%d%d%d)", "%1,")
    return minus .. int:reverse():gsub("^,", "") .. fraction
end

RegisterNetEvent('qb-gangmenu:client:OpenMenu', function()
    shownGangMenu = true
    
    -- Get account balance from banking system
    QBCore.Functions.TriggerCallback('qb-banking:server:getGangAccountBalance', function(balance)
        local gangMenu = {
            {
                header = Lang:t('headersgang.bsm') .. string.upper(PlayerGang.label),
                icon = 'fa-solid fa-circle-info',
                isMenuHeader = true,
            },
            {
                header = PlayerGang.label .. " Funds: $" .. format_int(balance),
                txt = "",
                icon = 'fa-solid fa-money-bill-wave',
                isMenuHeader = true,
            },
            {
                header = Lang:t('bodygang.manage'),
                txt = Lang:t('bodygang.managed'),
                icon = 'fa-solid fa-list',
                params = {
                    event = 'qb-gangmenu:client:ManageGang',
                }
            },
            {
                header = Lang:t('bodygang.hire'),
                txt = Lang:t('bodygang.hired'),
                icon = 'fa-solid fa-hand-holding',
                params = {
                    event = 'qb-gangmenu:client:HireMembers',
                }
            },
            {
                header = Lang:t('bodygang.storage'),
                txt = Lang:t('bodygang.storaged'),
                icon = 'fa-solid fa-box-open',
                params = {
                    isServer = true,
                    event = 'qb-gangmenu:server:stash',
                }
            },
            {
                header = Lang:t('bodygang.outfits'),
                txt = Lang:t('bodygang.outfitsd'),
                icon = 'fa-solid fa-shirt',
                params = {
                    event = 'qb-gangmenu:client:Warbobe',
                }
            }
        }

        for _, v in pairs(DynamicMenuItems) do
            gangMenu[#gangMenu + 1] = v
        end

        gangMenu[#gangMenu + 1] = {
            header = Lang:t('bodygang.exit'),
            icon = 'fa-solid fa-angle-left',
            params = {
                event = 'qb-menu:closeMenu',
            }
        }

        exports['qb-menu']:openMenu(gangMenu)
    end, PlayerGang.name)
end)
```

This code adds similar functionality to the gang menu, showing the gang's account balance to gang leaders.

## Working with Business Accounts

### Adding Money to a Business Account

When you need to add money to a business account (for example, from customer payments), use the following code:

```lua
exports['qb-banking']:AddMoney('businessname', amount, 'Reason for payment')
```

Example for a Burgershot payment:

```lua
exports['qb-banking']:AddMoney('burgershot', 150, 'Customer purchase')
```

### Getting Business Account Balance

If you need to check a business account balance in your scripts:

```lua
local accountBalance = exports['qb-banking']:GetAccountBalance('businessname')
```

## Testing the Implementation

1. Restart the affected resources:
   ```
   ensure qb-banking
   ensure qb-management
   ```

2. Log in as a boss of a job or leader of a gang

3. Access the boss/gang menu at the designated location

4. You should now see the "Company Funds" or "Gang Funds" display showing the current balance

## Troubleshooting

### Balance Shows as $0 When There Should Be Funds

1. Verify that the account exists in the `bank_accounts` table in your database
2. Check if the job/gang name matches exactly with the account name
3. Ensure the player has boss permissions for the job/gang

### Menu Doesn't Open or Errors Occur

1. Check server logs for error messages
2. Verify that the callbacks are registered correctly in `qb-banking`
3. Ensure all resource dependencies are properly started

### Function Not Found Errors

If you encounter errors about missing functions, ensure that the `GetAccount` function exists in your `qb-banking/server.lua` file. It should be defined in the exports section.

## Advanced: Customization

### Changing the Display Format

You can modify the `format_int` function to change how the balance is displayed. For example, to show only whole dollars without cents:

```lua
function format_int(number)
    local formatted = math.floor(number)
    local k
    while true do  
        formatted, k = string.gsub(formatted, "^(-?%d+)(%d%d%d)", '%1,%2')
        if k == 0 then break end
    end
    return formatted
end
```

### Adding Additional Financial Information

You can extend this implementation to show more information, such as recent transactions or financial statistics. This would require creating additional callbacks and modifying the menu structure.

## Conclusion

With this implementation, business owners and gang leaders can now view their account balances directly from the management menu, providing better financial visibility without having to visit a bank. This integration bridges the gap created by the separation of financial functionality from the management system in the QB-Core framework.
