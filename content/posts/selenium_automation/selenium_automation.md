---
title: "Leveraging Docker and Selenium for Efficient Appointment Checking"
date: "2024-04-01"
tags: ["docker", "selenium", "python", "git", "automation"]
#categories: [""]
series: ["docker"]
ShowToc: true
TocOpen: true
cover:
    image: "/posts/selenium_automation/cover.png"
    hiddenInSingle: true
summary: "This article explores an automation use-case with Selenium and Docker to periodically access a multi-page js form and check for available appointments."
---

In the fast-paced world of DevOps and automation, efficiency and reliability are paramount. Today, we delve into a Docker-based automation script that epitomizes these qualities. Our focus is a python script that utilizes SeleniumBase to navigate through multi-page JavaScript forms to identify available appointments.

This script not only operates seamlessly in the background, checking for appointments, but also notifies users via email, attaching screenshots upon successful detection. This process builds on our previous discussions on deploying applications using Docker, illustrating the breadth and versatility of containerization in modern development workflows.

Find the complete setup on instructions in the [GitHub repository](https://github.com/tbalza/cita-checker).

{{< rawhtml >}}
<video width="720" loop autoplay muted>
  <source src="/posts/selenium_automation/cita-check.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
{{< /rawhtml >}}

### Selenium: The Backbone of Web Automation

Selenium is more than just a tool; it's a comprehensive framework that enables developers to automate web browser interactions. Imagine you're testing a web application that sells concert tickets. Selenium allows you to write a script that mimics a user purchasing a ticket, from selecting a seat to filling out payment information. This ensures that your application behaves as expected across different web browsers and operating systems.

#### The Role of Selenium in DevOps and CI/CD

In a continuous deployment pipeline, automated testing is crucial. Selenium automates the testing of web applications, fitting perfectly into the CI/CD pipeline. For example, each time a developer commits changes to the codebase, Selenium tests can automatically run to verify that the web application still meets all functional requirements. This immediate feedback is vital for maintaining software quality in a fast-paced development environment.

#### Selenium vs. SeleniumBase: Enhancing Automation

While Selenium sets the stage for web automation, SeleniumBase takes it a step further by simplifying test case creation. With Selenium, you might write extensive code to navigate through a form and verify its contents. SeleniumBase encapsulates common patterns, such as waiting for elements to become visible or clicking elements, into simpler, more robust commands. This not only makes the test code cleaner but also more resilient to changes in the web application.

Using SeleniumBase, a script to check for available appointments might navigate through a series of dropdowns to select a location and service type with just a few lines of code, handling waits and clicks more efficiently than raw Selenium code.

With plain Selenium, selecting a dropdown would require this code,
```python
# Tramite select
wait.until(EC.visibility_of_element_located((By.ID, 'tramiteGrupo[0]'))) # Wait for second AJAX page to Load
dropdown2 = wait.until(EC.element_to_be_clickable((By.ID, 'tramiteGrupo[0]')))
dropdown2.click() # Ensure the dropdown is opened, might be necessary to trigger JavaScript
select_element2 = Select(dropdown2)
select_element2.select_by_visible_text('Toma de huella (expedicion de tarjeta). renovacion de tarjeta larga duracion y duplicado')
option = wait.until(EC.visibility_of_element_located((By.XPATH, "//select[@id='tramiteGrupo[0]']/option[contains(text(Toma de huella (expedicion de tarjeta)))]")))
driver.execute_script("arguments[0].click();",option)
```

Using SeleniumBase methods, the code above is simply,
```python
sb.select_option_by_text("#tramiteGrupo\\[0\\]", config['tramiteOptionText'])
```

#### Selenium IDE

Using tools such as Selenium IDE we can manually navigate the form completion process and record the CSS selectors, streamlining the conversion to SeleniumBase's API later on.

![ide](/posts/selenium_automation/selenium-ide.png)


#### Navigating with SeleniumBase

SeleniumBase offers the uc mode, which employs undetected-chromedriver to evade detection mechanisms that websites use to block automated browsers. This is particularly useful for appointment checking on sites with anti-bot measures.

### Docker's Role in Streamlining Development

Docker encapsulates applications and their environments into containers, ensuring consistency across development, testing, and production. In the context of web scraping and automation, Docker ensures that your SeleniumBase scripts run in an environment where all dependencies are met, regardless of the host system.

#### Dockerizing the Python Script

The `Dockerfile` starts from a Ubuntu base image, installs Chrome and the necessary drivers, and adds the script. This container can then be run on any system with Docker, without the need for additional setup.

```dockerfile
# SeleniumBase Docker Image
FROM ubuntu:18.04

WORKDIR /cita-checker/SeleniumBase

#=======================================
# Install Python and Basic Python Tools
#=======================================
RUN apt-get -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false update
RUN apt-get install -y python3 python3-pip python3-setuptools python3-dev python-distribute
RUN alias python=python3
RUN echo "alias python=python3" >> ~/.bashrc

#================
# Install Chrome
#================
RUN curl -sS -o - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - && \
    echo "deb http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google-chrome.list && \
    apt-get -yqq update && \
    apt-get -yqq install google-chrome-stable && \
    rm -rf /var/lib/apt/lists/*
```

With a submodule configuration other collaborators can fetch the script with `git clone --recurse-submodules https://github.com/tbalza/cita-checker.git` and create custom features, such as VNC access for manual interaction, while keeping the original SB repo untouched, streamlining development.

```dockerfile
#===========================================
# Install VNC Server, Window Manager, NoVNC
#===========================================
ENV DISPLAY=:99.0
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y \
    python-numpy \
    net-tools \
    x11vnc \
    xfce4 \
    xfce4-goodies \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /.novnc && cd /.novnc \
    && wget -qO- https://github.com/novnc/noVNC/archive/v1.0.0.tar.gz | tar xz --strip 1 -C $PWD \
    && mkdir /.novnc/utils/websockify \
    && wget -qO- https://github.com/novnc/websockify/archive/v0.6.1.tar.gz | tar xz --strip 1 -C /.novnc/utils/websockify \
    && ln -s /.novnc/vnc.html /.novnc/index.html

EXPOSE 5900
EXPOSE 6080
```
The steps for a developer to set up this submodule configuration are:
```bash
# Initialize main project repository
cd /cita-checker
git init
git add .
git commit -m "Initial commit"

# Add remote repository
git remote add origin https://github.com/tbalza/cita-checker
git push -u origin master

# Add sumbodule
git submodule add https: //github.com/seleniumbase/SeleniumBase SeleniumBase
git add .
git commit -m "Add seleniumbase submodule"
git push
```

With an entry point we can define the commands that will keep all our extra services running, tying up all the configuration. In our setup configuring Xvfb is crucial for our desktop environment and VNC to work alongside the SeleniumBase webdriver in headful mode.

```shell
#!/bin/bash

# Function to keep the Xvfb running
function keepUpScreen() {
    echo "Running keepUpScreen()"
    while true; do
        sleep 1
        if [ -z "$(pidof Xvfb)" ]; then
            echo "Xvfb is not running. Starting Xvfb..."
            Xvfb :99 -screen 0 1600x900x16 &
        fi
    done
}

# Configure and start Xvfb
export DISPLAY=:99.0
rm -f /tmp/.X99-lock &>/dev/null # remove the lock file for X server display number 99
Xvfb :99 -screen 0 1600x900x16 &

# Start xfce
startxfce4 &

# Start x11vnc without a password, accessible only from localhost
x11vnc -display :99 -rfbport 5900 -nopw -forever &

# Keep Xvfb running
keepUpScreen &

# Wait for any background processes to finish
wait $!

# Execute commands passed to the Docker container
exec "$@"
```

#### Advantages of a Containerized Approach

By dockerizing your appointment checking script, you not only ensure it runs in a consistent environment but also simplify deployment and scaling. If the script needs to run at multiple locations or at scale, Docker containers can be deployed across multiple machines or cloud instances seamlessly.

### Python Script for Appointment Checking

The provided Python script utilizes SeleniumBase's features to navigate a web form and check for appointments. It employs strategies like setting a random window size to mimic human behavior, running in headful mode to cater to the need for manual interaction, and using the uc mode to evade bot detection.

**A Closer Look at the Script's Logic**

#### Setting Browser Settings and Opening the Appointment Page
The script starts by opening the target URL with a random window size to avoid automation detection.

```python
def set_random_window_size(sb):
    min_width = 800  # Define minimum width
    max_width = 1600  # Define maximum width
    width = random.randint(min_width, max_width)  # Choose a random width within the range
    height = (width * 2) // 3  # Calculate the height based on a 3:2 aspect ratio
    sb.set_window_size(width, height)  # Set the window size with the calculated width and height
```

```python
def check_for_appointments():
    with SB(
            chromium_arg="--force-device-scale-factor=1",  # Needed to set window size
            browser="chrome",  # When running brave, leave it as chrome. Invokes chromedriver with default options
            # binary_location="/usr/bin/brave-browser",  # Uncomment for Brave Browser
            headed=True,  # Run tests in headed/GUI mode on Linux. (To have access to browser after)
            uc=True,  # Use undetected-chromedriver to evade bot-detection. Only works for Chrome/Brave
            use_auto_ext=False,  # Hide chrome's automation extension
            slow=True,  # Makes actions run slower
            incognito=True, # Enable Chromium's Incognito mode. Clear session cookies
    ) as sb:
        try:  # Clicking logic to get to appointment status
            set_random_window_size(sb)  # Adjust the browser window size (to avoid rate-limiting)
```

#### Navigation
and navigating through the form by selecting options and clicking buttons, demonstrating how SeleniumBase simplifies interaction with web elements.
```python
    ) as sb:
    try:  # Clicking logic to get to appointment status
    set_random_window_size(sb)  # Adjust the browser window size (to avoid rate-limiting)
    sb.open(config['url'])  # Fetches values.json
    sb.click("#form")
    sb.select_option_by_text("#form", "Barcelona")
    sb.click("#btnAceptar")
    sb.select_option_by_text("#tramiteGrupo\\[0\\]", config['tramiteOptionText'])  # Fetches values.json
    sb.click("#btnAceptar")
    sb.click("#btnEntrar")
    sb.type("#txtIdCitado", config['idCitadoValue'])  # Fetches values.json
    sb.type("#txtDesCitado", config['desCitadoValue'])  # Fetches values.json
    sb.select_option_by_text("#txtPaisNac", config['paisNacValue'])  # Fetches values.json
    sb.click("#btnEnviar")
    sb.click("#btnEnviar")

```

#### Checking for Availability
It then checks for text indicating whether appointments are available. If not found, it assumes availability and proceeds to take a screenshot for evidence, showcasing SeleniumBase's ability to easily interact with web pages and perform checks.
![selector](/posts/selenium_automation/cita-check-selector.png)
```python
# Checking for appointment availability
if sb.is_text_visible("En este momento no hay citas disponibles.",
                      "div.mf-main--content.ac-custom-content p"):
    logging.info("No available appointments. Trying again in 10 minutes.")
     return "retry"
else:
      # If the text is not found, assume an appointment is available
      sb.set_window_size(1280, 1024)  # Set correct resolution for screenshot
      sb.save_screenshot("/tmp/cita_disponible.png")  # Take a screenshot for the email attachment
      send_email("Cita Disponible Alert", "VNC to vnc://127.0.0.1:5900 to complete", attach_screenshot=True)
      logging.info("Appointments might be available. Keeping the browser open for manual check.")
      user_input = input("Type 'restart' and enter")
      if user_input.lower() == "restart": # SB Context manager quits automatically. workaround to maintain open
         return "manual_check_needed"

except Exception as e:
    logging.error(f"Encountered an error during the steps: {e}. Trying again in 10 minutes.")
     return "error"
```

#### Email Notifications
Upon finding an available appointment, the script sends an email notification with the screenshot attached. This demonstrates integrating SeleniumBase with other Python libraries for complete automation solutions.

```python
def send_email(subject, message, attach_screenshot=False):
    # Load email configuration from values.json
    with open('/tmp/values.json', 'r') as file:
        config = json.load(file)

    sender_email = config['sender_email']
    receiver_email = config['receiver_email']
    password = config['password']
    smtp_server = config['smtp_server']
    smtp_port = config['smtp_port']

    # Create the email message
    msg = EmailMessage()
    msg['Subject'] = subject
    msg['From'] = sender_email  # Fetches values.json
    msg['To'] = receiver_email  # Fetches values.json
    msg.set_content(message)

    # Attach the screenshot if required
    if attach_screenshot:
        screenshot_path = "/tmp/cita_disponible.png"
        if os.path.exists(screenshot_path):
            with open(screenshot_path, 'rb') as f:
                file_data = f.read()
                file_name = os.path.basename(screenshot_path)
                msg.add_attachment(file_data, maintype='image', subtype='png', filename=file_name)

    # Send the email
    try:
        with smtplib.SMTP_SSL(smtp_server, smtp_port) as smtp:  # Fetches values.json
            smtp.login(sender_email, password)  # Fetches values.json
            smtp.send_message(msg)
            logging.info("Email sent successfully!")
    except Exception as e:
        logging.error(f"Error sending email: {e}")
```
Once the script finds an available appointment you'll receive an email alert with a screenshot:
![alert](/posts/selenium_automation/email-alert.png)

You can set up a distinct alarm tone on your mobile phone for a particular sender to alert you during off-hours, as if you were on call.

### Conclusion

This detailed exploration showcases the power and versatility of using Docker and SeleniumBase for automating the task of checking for available appointments. Through practical examples and explanations, we've seen how these technologies can streamline the development and deployment of automation scripts, ensuring reliability, efficiency, and scalability in modern web application testing and automation tasks.