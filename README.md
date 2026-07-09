```python
#This source code is for discord owo bot automation (owo hunt and owo battle)
#but please note that this still might detect as spamming so please don't fall asleep while using this code

#I was banned for 3 months when I did it.

import pyautogui
import time
import random

#SETTINGS
msg1 = "wh"
msg2 = "wb"
min_interval = 15
max_interval = 25

print("Starting in 5 seconds... Click your target window.")
time.sleep(5)

swap = False  #controls message order

while True:
    if not swap:
        first, second = msg1, msg2
    else:
        first, second = msg2, msg1

    #Send first message
    pyautogui.write(first, interval=random.uniform(0.02, 0.07))
    pyautogui.press("enter")

    #Small human like pause
    time.sleep(random.uniform(0.5, 1.5))

    #Send second message
    pyautogui.write(second, interval=random.uniform(0.02, 0.07))
    pyautogui.press("enter")

    #Toggle order for next cycle
    swap = not swap

    #Random wait 15–25 seconds
    wait_time = random.uniform(min_interval, max_interval)
    time.sleep(wait_time)
```
