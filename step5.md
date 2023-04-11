# Task Resume test

## 1. DMS is not running
1. Stop the task.
2. Run update one person_dump and player.
```SQL
update person_dump set FirstName = 'G' where lastName = 'A'
update player set full_name = 'Shead, DeShawnmm4' where ID = 5157
```
3. Kafdrop should be no new messages.
4. Try to resume the task
5. DMS will handle the missed value, verify this in Kafdrop

## 2. DB source disconnect
1. Disable TCP 1433 in Windows Firewall.
Find Windows Defender Firewall

![image](https://user-images.githubusercontent.com/108851851/231229242-05d848d0-451d-4308-b5a1-02ae67abea12.png)

Click **All an apps or feature....**

![image](https://user-images.githubusercontent.com/108851851/231229402-c94f6e24-49a3-48a0-892a-16545f3b7e04.png)

Untick **Allow inbound TCP Port 1433**
Click **OK**

2. Run update one person_dump and player.
```SQL
update person_dump set FirstName = 'G' where lastName = 'A'
update player set full_name = 'Shead, DeShawnmm4' where ID = 5157
```
3. Check Kafdrop, no new message
4. Check CloudWatch, error exists

![Pasted image 20230411225152](https://user-images.githubusercontent.com/108851851/231229003-b93caabb-4ed5-48de-b196-6c476a447121.png)

Task terminate automatically after retry

![Pasted image 20230411225333](https://user-images.githubusercontent.com/108851851/231229039-1547543e-267e-4a8a-ac68-036bbd2f98c5.png)

5. When resume, it capture all the changes missed
