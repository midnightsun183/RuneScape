
Archives
The Invent with Python Blog
Wed 17 December 2014
Programming a Bot to Play the "Sushi Go Round" Flash Game
Posted by Al Sweigart in python   

Update 2015/01/08: If you attended my NCSS class on making a bot for this game, you can download the sushi.zip file which has all the code in bot5.py. Feel free to send questions to me at al@inventwithpython.com or tweet at me: @AlSweigart

This tutorial teaches how to write a bot that can automatically play the Flash game Sushi Go Round. The concepts in this tutorial can be applied to make bots that play other games as well. It's inspired by the How to Build a Python Bot That Can Play Web Games by Chris Kiehl. The primary improvement of this tutorial is it uses the cross-platform PyAutoGUI module to control the mouse and take screenshots. It is documented on its ReadTheDocs page.



Sushi Go Round is a resource management game similar to "Dine N Dash". You fill customer orders for different types of sushi and place them on the conveyor belt. Incoming customers may take other customer's sushi orders, forcing you to remake orders. Customers who wait too long to get orders end up leaving, costing you reputation. Angry customers can be placated with saki, but this bot does not make use of that feature. Ingredients will have to be ordered as they get low.

I've refined this bot so that it can successfully play all the way through the game, ending up with a score of about 38,000. The top scores are a little over 100,000, so there is still room for improvement with this bot. You can watch a YouTube video of the bot playing.



The source code and images for the bot can be downloaded here or viewed on GitHub.


Step 1 - Install PyAutoGUI and Download Images
First, install PyAutoGUI by downloading it from PyPI or installing it through the pip program that comes with Python 3. If you've installed PyAutoGUI before, use the -U option to update it to the latest version. This tutorial uses Python 3, though PyAutoGUI and the bot code work with either Python 2 or 3.

You can check if PyAutoGUI has been installed correctly by running import pyautogui in the interactive shell. Full documentation for PyAutoGUI is available at http://pyautogui.readthedocs.org.

PyAutoGUI provides basic image recognition by finding the screen coordinates of a provided image. To save yourself time of taking screenshots and making these images yourself, download them from the zip file here. All of the images should be stored in an /images folder:

       

Step 2 - Basic Setup
Create a sushigoround.py file and enter the following code:

#! python3
"""Sushi Go Round Bot
Al Sweigart al@inventwithpython.com @AlSweigart

A bot program to automatically play the Sushi Go Round flash game at http://miniclip.com/games/sushi-go-round/en/
"""

import pyautogui, time, os, logging, sys, random, copy

logging.basicConfig(level=logging.DEBUG, format='%(asctime)s.%(msecs)03d: %(message)s', datefmt='%H:%M:%S')
#logging.disable(logging.DEBUG) # uncomment to block debug log messages
This generic code sets up a shebang line, some comments to describe the program, imports for several modules, and setting up logging so that the logging.debug() function will output debug messages. (I explain why logging is preferable to print() calls in this blog post.

Step 3 - Constants Setup
Next, there are several variable constants set up for this program. I use constants as a way to detect typos; mistyping a string value as 'califrnia_roll' will cause bugs but Python won't directly point out the typo. Mistyping a variable name as CALIFRNIA_ROLL will cause Python to raise a NameError exception, letting you quickly fix it.

First, set up constants for each type of order in the game, as well as a tuple of all the orders:

# Food order constants (don't change these: the image filenames depend on these specific values)
ONIGIRI = 'onigiri'
GUNKAN_MAKI = 'gunkan_maki'
CALIFORNIA_ROLL = 'california_roll'
SALMON_ROLL = 'salmon_roll'
SHRIMP_SUSHI = 'shrimp_sushi'
UNAGI_ROLL = 'unagi_roll'
DRAGON_ROLL = 'dragon_roll'
COMBO = 'combo'
ALL_ORDER_TYPES = (ONIGIRI, GUNKAN_MAKI, CALIFORNIA_ROLL, SALMON_ROLL, SHRIMP_SUSHI, UNAGI_ROLL, DRAGON_ROLL, COMBO)
Next, set up constants for each of the ingredients and the recipes for each order. (This tutorial saves you from having to look up the recipe information in the game yourself.)

# Ingredient constants (don't change these: the image filenames depend on these specific values)
SHRIMP = 'shrimp'
RICE = 'rice'
NORI = 'nori'
ROE = 'roe'
SALMON = 'salmon'
UNAGI = 'unagi'
RECIPE = {ONIGIRI:         {RICE: 2, NORI: 1},
          CALIFORNIA_ROLL: {RICE: 1, NORI: 1, ROE: 1},
          GUNKAN_MAKI:     {RICE: 1, NORI: 1, ROE: 2},
          SALMON_ROLL:     {RICE: 1, NORI: 1, SALMON: 2},
          SHRIMP_SUSHI:    {RICE: 1, NORI: 1, SHRIMP: 2},
          UNAGI_ROLL:      {RICE: 1, NORI: 1, UNAGI: 2},
          DRAGON_ROLL:     {RICE: 2, NORI: 1, ROE: 1, UNAGI: 2},
          COMBO:           {RICE: 2, NORI: 1, ROE: 1, SALMON: 1, UNAGI: 1, SHRIMP: 1},}
Next are several constants and global variables used in the program. Although using global variables is generally a bad idea for making maintainable programs, this Python script is just a one off and I didn't want to make it complicated for newer programmers who don't yet know object-oriented programming concepts. The variables are described in the comments:

LEVEL_WIN_MESSAGE = 'win' # checkForGameOver() returns this value if the level has been won

# Settings
MIN_INGREDIENTS = 4 # if an ingredient gets below this value, order more
PLATE_CLEARING_FREQ = 8 # plates are cleared every this number of seconds, roughly
NORMAL_RESTOCK_TIME = 7 # the number of seconds it takes to restock inventory after ordering it (at normal speed, not express)
TIME_TO_REMAKE = 30 # if an order goes unfilled for this number of seconds, remake it

# Global variables
LEVEL = 1 # current level being played
INVENTORY = {SHRIMP: 5, RICE: 10,
             NORI: 10,  ROE: 10,
             SALMON: 5, UNAGI: 5}
GAME_REGION = () # (left, top, width, height) values coordinates of the game window
ORDERING_COMPLETE = {SHRIMP: None, RICE: None, NORI: None, ROE: None, SALMON: None, UNAGI: None} # unix timestamp when an ordered ingredient will have arrived
ROLLING_COMPLETE = 0 # unix timestamp of when the rolling of the mat will have completed
LAST_PLATE_CLEARING = 0 # unix timestamp of the last time the plates were cleared
LAST_GAME_OVER_CHECK = 0 # unix timestamp when we last checked for the Game Over or You Win messages
Note: a "unix timestamp" is the number of seconds since midnight, January 1, 1970. It is returned by the time.time() function. For example, if time.time() is called at 10:36:54.392 PM on December 16, 2014, it will return the value 1418798214.392. Ten seconds later it will return the value 1418798224.392. Subtracting these values is how the bot can measure how much time has passed.

And finally set up some global variables which will store coordinates for various things in the game window. The GAME_REGION variable stores the position of the Sushi Go Round game itself. Once that value has been acquired, the setupCoordinates() function (described later) will populate the values in these variables:

# various coordinates of objects in the game
GAME_REGION = () # (left, top, width, height) values coordinates of the entire game window
INGRED_COORDS = None
PHONE_COORDS = None
TOPPING_COORDS = None
ORDER_BUTTON_COORDS = None
RICE1_COORDS = None
RICE2_COORDS = None
NORMAL_DELIVERY_BUTTON_COORDS = None
MAT_COORDS = None
Step 4 - The main() Function
The main() ties together the code that runs from the very start of the bot program. It assumes that the Sushi Go Round game is visible on the screen and at the opening start screen (which has the flashing "PLAY" button).

Because PyAutoGUI takes control of the mouse, it can be hard to get the program's terminal window back in focus to shut it down. PyAutoGUI has a fail-safe feature; if you move the mouse cursor to the top-left corner of the screen PyAutoGUI will raise an exception and terminate the program.

def main():
    """Runs the entire program. The Sushi Go Round game must be visible on the screen and the PLAY button visible."""
    logging.debug('Program Started. Press Ctrl-C to abort at any time.')
    logging.debug('To interrupt mouse movement, move mouse to upper left corner.')
    getGameRegion()
    navigateStartGameMenu()
    setupCoordinates()
    startServing()
Step 5 - Image Recognition and Finding the Game Screen
Computer vision and optical character recognition (OCR) are major topics of computer science. The good thing about Sushi Go Round is that you don't need to know any of that because the game uses static 2D sprites. A california roll in the game will always look the same down to the pixel. So we can use PyAutoGUI's pixel-recognition functions to "see" where on the screen different objects are.

The pyautogui.locateOnScreen() function takes a string filename argument and returns a (left, top, width, height) tuple of integer coordinates for where the image is found on the screen. (The "left" value is the X-coordinate of the left edge, the "top" value is the Y-coordinate of the top edge. With the XY coordinates of the top-left corner and the width and height, you can completely describe the region.)

If the image isn't found on the screen, the function returns None. An optional region keyword argument can specify a smaller region of the screen to search (a smaller region means faster detection). Without a region specified, the entire screen is searched.

The location of the game window itself will be stored later in the GAME_REGION variable, which holds a (left, top, width, height) tuple of integers. (The (left, top, width, height) tuples are used throughout PyAutoGUI.) These values are visualized in the following graphic:



To find the game window, we search for the top-right corner image, which is stored in top_right_corner.png. (These image files are included in the zip file download.) Once this image is located on the screen, the GAME_REGION variable can be populated.

def imPath(filename):
    """A shortcut for joining the 'images/'' file path, since it is used so often. Returns the filename with 'images/' prepended."""
    return os.path.join('images', filename)


def getGameRegion():
    """Obtains the region that the Sushi Go Round game is on the screen and assigns it to GAME_REGION. The game must be at the start screen (where the PLAY button is visible)."""
    global GAME_REGION

    # identify the top-left corner
    logging.debug('Finding game region...')
    region = pyautogui.locateOnScreen(imPath('top_right_corner.png'))
    if region is None:
        raise Exception('Could not find game on screen. Is the game visible?')

    # calculate the region of the entire game
    topRightX = region[0] + region[2] # left + width
    topRightY = region[1] # top
    GAME_REGION = (topRightX - 640, topRightY, 640, 480) # the game screen is always 640 x 480
    logging.debug('Game region found: %s' % (GAME_REGION,))
Note that running PyAutoGUI's image recongition takes a relatively long time to execute (several hundred milliseconds). This can be mitigated by searching a smaller region instead of the entire screen. And since some objects will always appear in the same place in the game window, you can rely on using static integer coordinates instead of image recognition to find them.

Step 6 - Dealing with Coordinates
Many of the buttons in the game's user interface will always be in the same location. The coordinates for these buttons can be pre-programmed ahead of time. You could take a screenshot, paste the image into a graphics program such as Photoshop or MS Paint, and find all of the XY coordinates yourself. But this tutorial has done this step for you. The other coordinate global variables are populated in the setupCoordinates() function. Note that they are all relative to where the game window (whose coordinates are in GAME_REGION) is.

For example, the coordinates for the Salmon ordering button are set to (GAME_REGION[0] + 496, GAME_REGION[1] + 329). This is visualized in the following graphic:



def setupCoordinates():
    """Sets several of the coordinate-related global variables, after acquiring the value for GAME_REGION."""
    global INGRED_COORDS, PHONE_COORDS, TOPPING_COORDS, ORDER_BUTTON_COORDS, RICE1_COORDS, RICE2_COORDS, NORMAL_DELIVERY_BUTTON_COORDS, MAT_COORDS, LEVEL
    INGRED_COORDS = {SHRIMP: (GAME_REGION[0] + 40, GAME_REGION[1] + 335),
                     RICE:   (GAME_REGION[0] + 95, GAME_REGION[1] + 335),
                     NORI:   (GAME_REGION[0] + 40, GAME_REGION[1] + 385),
                     ROE:    (GAME_REGION[0] + 95, GAME_REGION[1] + 385),
                     SALMON: (GAME_REGION[0] + 40, GAME_REGION[1] + 425),
                     UNAGI:  (GAME_REGION[0] + 95, GAME_REGION[1] + 425),}
    PHONE_COORDS = (GAME_REGION[0] + 560, GAME_REGION[1] + 360)
    TOPPING_COORDS = (GAME_REGION[0] + 513, GAME_REGION[1] + 269)
    ORDER_BUTTON_COORDS = {SHRIMP: (GAME_REGION[0] + 496, GAME_REGION[1] + 222),
                           UNAGI:  (GAME_REGION[0] + 578, GAME_REGION[1] + 222),
                           NORI:   (GAME_REGION[0] + 496, GAME_REGION[1] + 281),
                           ROE:    (GAME_REGION[0] + 578, GAME_REGION[1] + 281),
                           SALMON: (GAME_REGION[0] + 496, GAME_REGION[1] + 329),}
    RICE1_COORDS = (GAME_REGION[0] + 543, GAME_REGION[1] + 294)
    RICE2_COORDS = (GAME_REGION[0] + 545, GAME_REGION[1] + 269)

    NORMAL_DELIVERY_BUTTON_COORDS = (GAME_REGION[0] + 495, GAME_REGION[1] + 293)

    MAT_COORDS = (GAME_REGION[0] + 190, GAME_REGION[1] + 375)

    LEVEL = 1
This tutorial does the tedious coordinate-finding work for you, but if you ever need to do it yourself you can use the pyautogui.displayMousePosition() function to display the XY coordinates of the mouse cursor. The coordinates update as you move the mouse, so you can move the mouse to the desired location on the screen, then view its current coordinates. The displayMousePosition() function also shows that pixel's RGB color value.



You can also pass X and Y offsets to the function to set the (0, 0) origin somewhere besides the top-left corner of the screen. When you are done, press Ctrl-C to return from the function.

Step 7 - Controlling the Mouse
PyAutogui has a pyautogui.click() function that can be passed a (x, y) tuple argument of the coordinates on the screen to click on. Often you can use the return value of pyautogui.locateCenterOnScreen() for this argument. The click is done immediately, but the optional duration keyword argument will specify the number of seconds PyAutoGUI spends moving to the (x, y) coordinate. A small delay makes it easier to visually follow along with the bot's clicking.

Since the SKIP button in the game is flashing, the pyautogui.locateCenterOnScreen(imPath('skip_button.png'), region=GAME_REGION) call might not be able to find it if it is flashing in the other color than skip_button.png has. This is why there is a while loop that keeps searching until it finds it. Remember, locateCenterOnScreen() will return None if it can't find the image on the screen.

def navigateStartGameMenu():
    """Performs the clicks to navigate form the start screen (where the PLAY button is visible) to the beginning of the first level."""
    # Click on everything needed to get past the menus at the start of the game.

    # click on Play
    logging.debug('Looking for Play button...')
    while True: # loop because it could be the blue or pink Play button displayed at the moment.
        pos = pyautogui.locateCenterOnScreen(imPath('play_button.png'), region=GAME_REGION)
        if pos is not None:
            break
    pyautogui.click(pos, duration=0.25)
    logging.debug('Clicked on Play button.')

    # click on Continue
    pos = pyautogui.locateCenterOnScreen(imPath('continue_button.png'), region=GAME_REGION)
    pyautogui.click(pos, duration=0.25)
    logging.debug('Clicked on Continue button.')

    # click on Skip
    logging.debug('Looking for Skip button...')
    while True: # loop because it could be the yellow or red Skip button displayed at the moment.
        pos = pyautogui.locateCenterOnScreen(imPath('skip_button.png'), region=GAME_REGION)
        if pos is not None:
            break
    pyautogui.click(pos, duration=0.25)
    logging.debug('Clicked on Skip button.')

    # click on Continue
    pos = pyautogui.locateCenterOnScreen(imPath('continue_button.png'), region=GAME_REGION)
    pyautogui.click(pos, duration=0.25)
    logging.debug('Clicked on Continue button.')
Once the buttons at the start of the game have been navigated past, the main part of the bot code can begin.

Step 8 - The startServing() Function
The startServing() function handles all of the main game play. This includes:

Seeing which orders are being requested from customers.
Creating dishes to fill orders.
Ordering more ingredients when running low on them.
Checking if customers haven't received their orders in a long time and remaking those orders.
Clearing finished plates.
Checking if ordered ingredients have arrived.
Checking if the game has been lost or if the level has been won.
Much of this functionality is passed on to other functions, which are explained later. The first part of startServing() sets up all the variables for a new game:

def startServing():
    """The main game playing function. This function handles all aspects of game play, including identifying orders, making orders, buying ingredients and other features."""
    global LAST_GAME_OVER_CHECK, INVENTORY, ORDERING_COMPLETE, LEVEL

    # Reset all game state variables.
    oldOrders = {}
    backOrders = {}
    remakeOrders = {}
    remakeTimes = {}
    LAST_GAME_OVER_CHECK = time.time()
    ORDERING_COMPLETE = {SHRIMP: None, RICE: None, NORI: None,
                         ROE: None, SALMON: None, UNAGI: None}
The next part scans the region where customers orders are displayed:



This is handled by the getOrders() function. Since you'll be repeatedly scanning for the orders but don't want to remake orders each time you scan them, you need to find out which new orders have appeared since the last scan, and which orders have disappeared since the last scan (either because the customer is eating or has left). This is handled by the getOrdersDifference().

The keys in these "orders dictionaries" will be the (left, top, width, height) tuple of where the order image was identified. This is just to keep the customers distinct. The values will be the strings in one of the ingredient constants like CALIFORNIA_ROLL or ONIGIRI. The remakeTimes dictionary has similar keys, but it's values are unix timestamps (returned from time.time()) for when the dish should be remade if it is still being requested by the customer.

    while True:
        # Check for orders, see which are new and which are gone since last time.
        currentOrders = getOrders()
        added, removed = getOrdersDifference(currentOrders, oldOrders)
        if added != {}:
            logging.debug('New orders: %s' % (list(added.values())))
            for k in added:
                remakeTimes[k] = time.time() + TIME_TO_REMAKE
        if removed != {}:
            logging.debug('Removed orders: %s' % (list(removed.values())))
            for k in removed:
                del remakeTimes[k]
The next part goes through the remakeTimes dictionary and checks if any of those timestamps are before the current time (as returned by time.time()). In that case, add this order to the remakeOrders dictionary (which is similar but separate to the added orders dictionary of new orders.)

        # Check if the remake times have past, and add those to the remakeOrders dictionary.
        for k, remakeTime in copy.copy(remakeTimes).items():
            if time.time() > remakeTime:
                remakeTimes[k] = time.time() + TIME_TO_REMAKE # reset remake time
                remakeOrders[k] = currentOrders[k]
                logging.debug('%s added to remake orders.' % (currentOrders[k]))
Next, the program loops through all the newly requested orders in the added dictionary and attempts to make them. The makeOrder() function (described later) will return None if the order was successfully made. Otherwise, it will return a string of the ingredient it doesn't have enough of. In that case, orderIngredient() (described later) is called and the order is placed on the backOrders dictionary.

        # Attempt to make the order.
        for pos, order in added.items():
            result = makeOrder(order)
            if result is not None:
                orderIngredient(result)
                backOrders[pos] = order
                logging.debug('Ingredients for %s not available. Putting on back order.' % (order))
When a customer picks up their meal, they'll spend a few seconds eating and then leave behind a dirty dish. Click this dirty dish so that a new customer will take that seat. Instead of using image recognition (which is slow), the bot can just occassionally click on all six plates. There's no penalty to clicking on the plate when the customer is eating or if there is no plate at that seat.

Since this doesn't need to be done frequently, the if random.randint(1, 10) == 1 or time.time() - PLATE_CLEARING_FREQ > LAST_PLATE_CLEARING: statement will only do this roughly once every 10 iterations, or if more than 6 seconds (the value in PLATE_CLEARING_FREQ) has passed since the last plate clearing (according to the timestamp in LAST_PLATE_CLEARING).

The actual clicking is done by the clickOnPlates() function, described later.

        # Clear any finished plates.
        if random.randint(1, 10) == 1 or time.time() - PLATE_CLEARING_FREQ > LAST_PLATE_CLEARING:
            clickOnPlates()
It's easy to keep track of ingredient quanties as you make dishes. Each level starts with a set quantity of ingredients, encoded in the INVENTORY = {SHRIMP: 5, RICE: 10, NORI: 10, ROE: 10, SALMON: 5, UNAGI: 5} lines. You can subtract values in the INVENTORY dictionary as you use ingredients.

When you order ingredients they don't arrive immediately. If you used image recognition for the numbers in the lower left corner of the game window it would require grabbing tons of screenshots and slow down the bot. Instead, since orders take less than 7 seconds to arrive (which is why the NORMAL_RESTOCK_TIME is set to 7), the ORDERING_COMPLETE can store unix timestamps for 7 seconds in the future when INVENTORY can be updated.

The values in ORDERING_COMPLETE are None if they are not currently being ordered and delivered.

The next part of code checks if the current time (as returned by time.time()) has passed. If so, the INVENTORY dictionary's values are updated.

        # Check if ingredient orders have arrived.
        updateInventory()

        # Go through and see if any back orders can be filled.
        for pos, order in copy.copy(backOrders).items():
            result = makeOrder(order)
            if result is None:
                del backOrders[pos] # remove from back orders
                logging.debug('Filled back order for %s.' % (order))
The remakeOrders dictionary contains orders that (for whatever reason) didn't make it to the customer. This code is similar to the previous dish-making code:

        # Go through and see if any remake orders can be filled.
        for pos, order in copy.copy(remakeOrders).items():
            if pos not in currentOrders:
                del remakeOrders[pos]
                logging.debug('Canceled remake order for %s.' % (order))
                continue
            result = makeOrder(order)
            if result is None:
                del remakeOrders[pos] # remove from remake orders
                logging.debug('Filled remake order for %s.' % (order))
The next part checks the INVENTORY dictionary to see if there is less than 4 (the value stored in the MIN_INGREDIENTS constant) of any ingredient. If so, more is ordered by calling the orderIngredient() function (described later). This isn't something that needs to be checkeded frequently, so it is enclosed in an if statement that runs 1 in 5 times:

        if random.randint(1, 5) == 1:
            # order any ingredients that are below the minimum amount
            for ingredient, amount in INVENTORY.items():
                if amount < MIN_INGREDIENTS:
                    orderIngredient(ingredient)
The next part checks for the "You Win" or "You Fail" message. Since this check doesn't need to happen frequently at all, the if time.time() - 12 > LAST_GAME_OVER_CHECK: statement only runs it once every 12 seconds. LAST_GAME_OVER_CHECK is updated with the current time from time.time() every time the check is performed. The checkForGameOver() function (described later) terminates the program if the player has lost, or returns the string in the LEVEL_WIN_MESSAGE constant if the level has been beaten.

If the level has been beaten, the code resets many of the variables:

        # check for the "You Win" or "You Fail" messages
        if time.time() - 12 > LAST_GAME_OVER_CHECK:
            result = checkForGameOver()
            if result == LEVEL_WIN_MESSAGE:
                # player has completed the level

                # Reset inventory and orders.
                INVENTORY = {SHRIMP: 5, RICE: 10,
                             NORI: 10, ROE: 10,
                             SALMON: 5, UNAGI: 5}
                ORDERING_COMPLETE = {SHRIMP: None, RICE: None,
                                     NORI: None, ROE: None,
                                     SALMON: None, UNAGI: None}
                backOrders = {}
                remakeOrders = {}
                currentOrders = {}
                oldOrders = {}
Also, the bot will let the user view the end-level stats for 5 seconds before clicking Continue to move on to the next level.

                logging.debug('Level %s complete.' % (LEVEL))
                LEVEL += 1
                time.sleep(5) # give another 5 seconds to tally score

                # Click buttons to continue to next level.
                pos = pyautogui.locateCenterOnScreen(imPath('continue_button.png'), region=GAME_REGION)
                pyautogui.click(pos, duration=0.25)
                logging.debug('Clicked on Continue button.')
                pos = pyautogui.locateCenterOnScreen(imPath('continue_button.png'), region=GAME_REGION)
For every level except for the last, there will be a second Continue button to click on. On the last level, the bot doesn't do this so the user can view the game ending.

                if LEVEL <= 7: # click the second continue if the game isn't finished.
                    pyautogui.click(pos, duration=0.25)
                    logging.debug('Clicked on Continue button.')

        oldOrders = currentOrders
Step 9 - Clearing the Plates
The clickOnPlates() function blindly clicks on all six places where there could be a dirty plate to clean up. This works because there is no penalty for clicking when there isn't a dirty plate, freeing the bot from having to use performance-costly image recognition to first identify the dirty plate.

The LAST_PLATE_CLEARING global variable contains the timestamp of the last time the plates were cleared. This is used to determine if a long time has passed since the plates have been cleared.

def clickOnPlates():
    """Clicks the mouse on the six places where finished plates will be flashing. This function does not check for flashing plates, but simply clicks on all six places.

    Sets LAST_PLATE_CLEARING to the current time."""
    global LAST_PLATE_CLEARING

    # just blindly click on all the places where a plate should be
    for i in range(6):
        pyautogui.click(83 + GAME_REGION[0] + (i * 101), GAME_REGION[1] + 203)
    LAST_PLATE_CLEARING = time.time()
Step 10 - Getting the Orders from Customers
Customer orders appear as dish images in word bubbles above their heads. PyAutoGUI's image recognition can be used to scan this region of the game screen and match them to the images in california_roll_order.png, gunkan_maki_order.png, onigiri_order.png, and so on:

  

The getOrders() function finds all the current orders being requested. Figuring out which of these orders have been seen before (and possible already fulfilled) is done by the getOrdersDifference() function, explained in the next section.

def getOrders():
    """Scans the screen for orders being made. Returns a dictionary with a (left, top, width, height) tuple of integers for keys and the order constant for a value.

    The order constants are ONIGIRI, GUNKAN_MAKI, CALIFORNIA_ROLL, SALMON_ROLL, SHRIMP_SUSHI, UNAGI_ROLL, DRAGON_ROLL, COMBO."""
    orders = {}
    for orderType in (ALL_ORDER_TYPES):
        allOrders = pyautogui.locateAllOnScreen(imPath('%s_order.png' % orderType), region=(GAME_REGION[0] + 32, GAME_REGION[1] + 46, 558, 44))
        for order in allOrders:
            orders[order] = orderType
    return orders

The newOrders and oldOrders parameters have values that were returned from different calls to getOrders(). The orders that are new to newOrders are returned in the added dictionaries and the orders that have since been removed are returned in the removed dictionary.

def getOrdersDifference(newOrders, oldOrders):
    """Finds the differences between the orders dictionaries passed. Return value is a tuple of two dictionaries.

    The first dictionary is the "added" dictionary of orders added to newOrders since oldOrders. The second dictionary is the "removed" dictionary of orders in oldOrders but removed in newOrders.

    Each dictionary has (left, top, width, height) for keys and an order constant for a value."""
    added = {}
    removed = {}

    # find all orders in newOrders that are new and not found in oldOrders
    for k in newOrders:
        if k not in oldOrders:
            added[k] = newOrders[k]
    # find all orders in oldOrders that were removed and not found in newOrders
    for k in oldOrders:
        if k not in newOrders:
            removed[k] = oldOrders[k]

    return added, removed
Step 11 - Making the Orders
Creating a dish involves clicking on the correct ingredients and then clicking on the rolling mat. The dish is place on a conveyor belt and eventually makes its way to a customer. The first part of the function is just the docstring and a few global statements.

def makeOrder(orderType):
    """Does the mouse clicks needed to create an order.

    The orderType parameter has the value of one of the ONIGIRI, GUNKAN_MAKI, CALIFORNIA_ROLL, SALMON_ROLL, SHRIMP_SUSHI, UNAGI_ROLL, DRAGON_ROLL, COMBO constants.

    The INVENTORY global variable is updated in this function for orders made.

    The return value is None for a successfully made order, or the string of an ingredient constant if that needed ingredient is missing."""
    global ROLLING_COMPLETE, INGRED_COORDS, INVENTORY
The next part of the code ensures that the bot doesn't try to make a dish when it's not possible. One of the reasons is that the mat is still in the process of rolling the previous dish. Also, a dish will stay on the mat after being rolled if there isn't space on the conveyor belt. (This happens when orders aren't picked up by a customer.)

This is an important check: if the bot starts clicking on ingredients when the rolling mat is occupied, then the bot's INVENTORY dictionary will contain an inaccurate count of ingredients, leading to other mistakes and eventually losing the game.

The mat takes about 1.5 seconds to complete, which is why the ROLLING_COMPLETE = time.time() + 1.5 is run after clicking the mat to begin rolling it. This is a simple check that doesn't take long to run.

But if the conveyor belt is occupied, then the previous order could still be on the mat waiting for a clear space on the belt. After performing the cheap ROLLING_TIME check, the bot then uses pyautogui.locateOnScreen() to see if the mat is clear. This guarantees that when the bot starts clicking on ingredients they will be moved to the mat.

    # wait until the mat is clear. The previous order could still be there if the conveyor belt has been full or the mat is currently rolling.
    while time.time() < ROLLING_COMPLETE and pyautogui.locateOnScreen(imPath('clear_mat.png'), region=(GAME_REGION[0] + 115, GAME_REGION[1] + 295, 220, 175)) is None:
        time.sleep(0.1)
But before the bot begins making a dish, it consults the INVENTORY dictionary to make sure it has enough of each of the ingredients in the dish's recipe. The recipes for each dish are held in the RECIPES constant.

If there isn't enough of an ingredient, the string in the ingredient's constant is returned. This information will be used to place an order for more of that ingredient.

    # check that all ingredients are available in the inventory.
    for ingredient, amount in RECIPE[orderType].items():
        if INVENTORY[ingredient] < amount:
            logging.debug('More %s is needed to make %s.' % (ingredient, orderType))
            return ingredient
Once it has been confirmed there is enough of each ingredient, calls to pyautogui.click() cause the bot to click on the appropriate ingredients, then click on the mat to roll it. The ROLLING_COMPLETE global variable is updated with the timestamp of when the mat rolling will be finished (1.5 seconds in the future) so the next dish can be prepared.

Just before the rolling mat is clicked though, the bot will check the conveyor belt for any excess dishes that weren't picked up by customers. While this is a waste of ingredients, these excess dishes tend to pile up (especially in the later levels), and slow down the bot from serving dishes to the point that it could lose the game. The call to findAndClickPlatesOnBelt() (described next) gets rid of any excess dishes that happen to be on the conveyor belt.

    # click on each of the ingredients
    for ingredient, amount in RECIPE[orderType].items():
        for i in range(amount):
            pyautogui.click(INGRED_COORDS[ingredient], duration=0.25)
            INVENTORY[ingredient] -= 1
    findAndClickPlatesOnBelt() # get rid of any left over meals on the conveyor belt, which may stall this meal from being loaded on the belt
    pyautogui.click(MAT_COORDS, duration=0.25) # click the rolling mat to make the order
    logging.debug('Made a %s order.' % (orderType))
    ROLLING_COMPLETE = time.time() + 1.5 # give the mat enough time (1.5 seconds) to finish rolling before being used again

Sometimes other customers will swoop in and grab dishes that were intended for other customers. To prevent the customers from leaving, the remakeTimes dictionary keeps track of when orders will need to be remade. However, if the customer leaves before picking up this remade dish, then it will circle endlessly on the conveyor belt. Eventually these excess dishes can pile up and stall new dishes from being added. The findAndClickPlatesOnBelt() will scan the conveyor belt next to the rolling mat for any dishes and click on them to remove them.

It does this by identifying the pink, blue, or red coloring of the plates. These colors are stored in the pink_plate_color.png, blue_plate_color.png, red_plate_color.png images.

def findAndClickPlatesOnBelt():
    """Find any plates on the conveyor belt that can be removed and click on them to remove them. This will get rid of excess orders."""
    for color in ('pink', 'blue', 'red'):
        result = pyautogui.locateCenterOnScreen(imPath('%s_plate_color.png' % (color)), region=(GAME_REGION[0] + 343, GAME_REGION[1] + 300, 50, 80))
        if result is not None:
            pyautogui.click(result)
            logging.debug('Clicked on %s plate on belt at X: %s Y: %s' % (color, result[0], result[1]))
Step 12 - Ordering More Ingredients
When the bot needs more ingredients, it must navigate the menu buttons on the phone in the lower right corner of the game window. This tutorial has already figured out these coordinates for you, but you could use the pyautogui.displayMousePosition() function to find coordinates for any pixel on the screen.

The first step is to click on the phone:

def orderIngredient(ingredient):
    """Do the clicks to purchase an ingredient. If successful, the ORDERING_COMPLETE dictionary is updated for when the ingredients will arive and INVENTORY can be updated. (This is handled in the updateInventory() function.)"""
    logging.debug('Ordering more %s (inventory says %s left)...' % (ingredient, INVENTORY[ingredient]))
    pyautogui.click(PHONE_COORDS, duration=0.25)
The next part of the code handles pressing the buttons to order rice. The timestamp in ORDERING_COMPLETE is checked to make sure a previous rice order hasn't been made. Then the screen is scanned for the cant_afford_rice.png image. The rice ordering button takes on a dimmed appearance if the player does not have enough money to order rice. In this case, the function returns.



    if ingredient == RICE and ORDERING_COMPLETE[RICE] is None:
        # Order rice.
        pyautogui.click(RICE1_COORDS, duration=0.25)

        # Check if we can't afford the rice
        if pyautogui.locateOnScreen(imPath('cant_afford_rice.png'), region=(GAME_REGION[0] + 498, GAME_REGION[1] + 242, 90, 75)):
            logging.debug("Can't afford rice. Canceling.")
            pyautogui.click(GAME_REGION[0] + 585, GAME_REGION[1] + 335, duration=0.25) # click cancel phone button
            return
Otherwise, the bot will click on the rice button. The code doesn't update INVENTORY quite yet, as it will take time for the rice to be delivered. (For simplicity, the bot will never use the Express Delivery option, though you could certainly build in the intelligence to do that.) The expected delivery time is set in the ORDERING_COMPLETE dictionary.

The INVENTORY will be updated with the new quantity in the updateInventory() function.

        # Purchase the rice
        pyautogui.click(RICE2_COORDS, duration=0.25)
        pyautogui.click(NORMAL_DELIVERY_BUTTON_COORDS, duration=0.25)
        ORDERING_COMPLETE[RICE] = time.time() + NORMAL_RESTOCK_TIME
        logging.debug('Ordered more %s' % (RICE))
        return
The next part does something similar except it navigates the phone menu for the non-rice ingredients. The same general logic is used; if an ingredient isn't affordable, the dimmed button image will inform the bot that it can't be purchased.

    elif ORDERING_COMPLETE[ingredient] is None:
        # Order non-rice ingredient.
        pyautogui.click(TOPPING_COORDS, duration=0.25)

        # Check if we can't afford the ingredient
        if pyautogui.locateOnScreen(imPath('cant_afford_%s.png' % (ingredient)), region=(GAME_REGION[0] + 446, GAME_REGION[1] + 187, 180, 180)):
            logging.debug("Can't afford %s. Canceling." % (ingredient))
            pyautogui.click(GAME_REGION[0] + 597, GAME_REGION[1] + 337, duration=0.25) # click cancel phone button
            return
Note that the bot doesn't have to keep track of how much money it has. It only needs to know if it can afford an ingredient or not. For simplicity, the "normal delivery" option is always used instead of the costly "express delivery" option:

        # Order the ingredient
        pyautogui.click(ORDER_BUTTON_COORDS[ingredient], duration=0.25)
        pyautogui.click(NORMAL_DELIVERY_BUTTON_COORDS, duration=0.25)
        ORDERING_COMPLETE[ingredient] = time.time() + NORMAL_RESTOCK_TIME
        logging.debug('Ordered more %s' % (ingredient))
        return
If it turns out the bot couldn't afford the desired ingredient, the next part will close the phone menu by clicking on the coordinates for the "hang up" button:

    # The ingredient has already been ordered, so close the phone menu.
    pyautogui.click(GAME_REGION[0] + 589, GAME_REGION[1] + 341) # click cancel phone button
    logging.debug('Already ordered %s.' % (ingredient))
The updateInventory() function is frequently called to check if the timestamps in ORDERING_COMPLETE indicate that the ordered ingredients have arrived. In that case, it updates INVENTORY with the new values. Shrimp, unagi, and salmon deliveries always add 5. Nori, roe, and rice deliveries always add 10.

def updateInventory():
    """Check if any ordered ingredients have arrived by looking at the timestamps in ORDERING_COMPLETE.
    Update INVENTORY global variable with the new quantities."""
    for ingredient in INVENTORY:
        if ORDERING_COMPLETE[ingredient] is not None and time.time() > ORDERING_COMPLETE[ingredient]:
            ORDERING_COMPLETE[ingredient] = None
            if ingredient in (SHRIMP, UNAGI, SALMON):
                INVENTORY[ingredient] += 5
            elif ingredient in (NORI, ROE, RICE):
                INVENTORY[ingredient] += 10
            logging.debug('Updated inventory with added %s:' % (ingredient))
            logging.debug(INVENTORY)
Step 13 - Checking for the End of the Level
The bot can tell when the level is over by doing image recognition for the you_win.png or you_fail.png images. If the game is over, the program terminates by calling sys.exit(). If the level has been beaten, the bot clicks on the "You Win" window so that it immediately tallies the stats and returns the string in the LEVEL_WIN_MESSAGE.

def checkForGameOver():
    """Checks the screen for the "You Win" or "You Fail" message.

    On winning, returns the string in LEVEL_WIN_MESSAGE.

    On losing, the program terminates."""

    # check for "You Win" message
    result = pyautogui.locateOnScreen(imPath('you_win.png'), region=(GAME_REGION[0] + 188, GAME_REGION[1] + 94, 262, 60))
    if result is not None:
        pyautogui.click(pyautogui.center(result))
        return LEVEL_WIN_MESSAGE

    # check for "You Fail" message
    result = pyautogui.locateOnScreen(imPath('you_failed.png'), region=(GAME_REGION[0] + 167, GAME_REGION[1] + 133, 314, 39))
    if result is not None:
        logging.debug('Game over. Quitting.')
        sys.exit()
Finally, the main() function is called if this script is being run (as opposed to imported as a module):

if __name__ == '__main__':
    main()
That's it!
If you download and run the bot, it doesn't have much problem beating all 7 levels of Sushi Go Round. But as you can tell from the leaderboard, if doesn't compare to the best human players. Feel free to modify the code to try to improve the performance of the bot.

Sushi Go Round is a good candidate for creating a bot since it's 2D sprite graphics are static and easy to identify from screenshots. If you find any other similar such games, please list them in the comments below! I've found that the "Skill" genre on Newgrounds, "puzzle", "gathering", and "rhythm" games are usually bot-friendly since their images are easier to identify. (There's a list of Flash game genres here.) Look for ones that have retro pixel graphics. In particular, these games seem like good candidates for making bots:

Color Finger
Cardinal
Sagacious Saga (helps if you know the floodfill algorithm)
Apple Catcher
Cold Runner
GatherX
Next Avoider
Dodge Copter
Rockocroco
Holy Stomping
And some trickier games, but still possible to make bots for:

1
Golden Duel
Homestar Runner: Marshie's Malloween Mix-Up
Twin Cube
Rollasaurus
Hyperwall
Update: roddds on Reddit posted a link to his own Sushi Go Round bot he had made previously. He also has code and a video for a Diamond Dash-playing bot.

Comments
