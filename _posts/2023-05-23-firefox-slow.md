---
layout: post
title: How to fix "A Web Page Is Slowing Down Your Browser" on Firefox?
---

Accessing any web page stuck with warning "A Web Page Is Slowing Down Your Browser. What would you like to do?"

<img width="1037" alt="Screenshot 2023-05-21 at 14 20 01" src="https://github.com/Meiyan-Zheng/meiyanblog/assets/30589773/0809b01d-8478-43ec-aa2b-fb47dfa9daf8">

Try turning off this Firefox feature to see if the “A web page is slowing down your browser” warning message appears again.
1. In Firefox, type `about:config` into the address bar and press the Enter key on your keyboard. You’ll be redirected to a settings page.

2. If prompted, click on the Accept the Risk and Continue button.
<img width="1440" alt="Screenshot 2023-05-21 at 14 22 14" src="https://github.com/Meiyan-Zheng/meiyanblog/assets/30589773/d8535624-e9f9-4c99-bd88-5226c12287ce">

3. Use the search bar on top of the page to look for processHang. You should see 2 options displayed after successfully searching.
<img width="1440" alt="Screenshot 2023-05-21 at 14 22 52" src="https://github.com/Meiyan-Zheng/meiyanblog/assets/30589773/2d676c8d-4ffd-48c1-aba5-f861f95d97e8">

4. Doubl click both `dom.ipc.reportProcessHangs` and `dom.ipc.processHangMonitor` then those parameters will become false.

5. Restart Firefox.

### Reference Links:
[What Is the “A Web Page Is Slowing Down Your Browser” Warning in Firefox?
](https://softwarekeep.com/help-center/what-is-the-a-web-page-is-slowing-down-your-browser-warning-in-firefox)
