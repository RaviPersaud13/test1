# Vending Machine Main Program
# Student Name: Ravi Persaud
# Student ID : 100760022

import PySimpleGUI as sg
from time import sleep

# Define the initial states for the Vending Machine
class State:
    """A class used to represent the state of the Vending Machine."""
    def __init__(self, name):
        """
        Initialize the state.

        Parameters:
        name (str): The name of the state.
        """
        # Takes a name as a parameter
        self.name = name

    def on_event(self, event):
        """Handle events that are delegated to this State."""
        pass

    def on_enter(self):
        """Handle any initialization upon entering this State."""
        pass

    def on_exit(self):
        """Handle any cleanup upon exiting this State."""
        pass

class WaitingState(State):
    """State representing the Vending Machine waiting for user interaction."""
    pass

class AddCoinsState(State):
    """State representing the Vending Machine during the coin addition process."""
    pass
        
class DeliverProductState(State):
    """State representing the Vending Machine when it is delivering a product."""
    pass

class CountChangeState(State):
    """State representing the Vending Machine when it is counting change to return."""
    pass

# Define the Vending Machine class
class VendingMachine:
    """A class to represent a Vending Machine."""
    def __init__(self, window):
        """
        Initialize the Vending Machine with a GUI window.
        
        """
        self.state = WaitingState("Waiting")
        self.window = window
        self.total_inserted = 0
        self.selected_product = None
        self.change_to_return = 0
        self.product_stock = {
            '-Gum-': 5,
            '-Pop-': 5,
            '-Chips-': 5,
            '-Chocolate-': 5,
            '-Beer-': 5
        }
        self.hardware_present = False

        try:
            from gpiozero import Button, Servo
            self.servo = Servo(17)
            self.button = Button(5)
            self.button.when_pressed = self.return_coins
            self.hardware_present = True
        except (ModuleNotFoundError, ImportError):
            print("Not on a Raspberry Pi or gpiozero not installed.")
            self.servo = None
            self.button = None

    def return_coins(self):
        """Return coins to the user and update the display."""
        self.change_to_return = self.total_inserted
        self.total_inserted = 0
        self.update_display()
        self.window['-STATUS-'].update('Transaction cancelled. Please take your coins.')

    def add_coin(self, coin_value):
        """
        Add a coin to the Vending Machine and update the display.

        """
        self.total_inserted += coin_value
        self.update_display()
        self.window['-STATUS-'].update('Please select a product or add more coins.')

    def select_product(self, product_key, product_price):
        """
        Process the selection of a product and update the display.

        """
        if self.product_stock[product_key] > 0:
            if self.total_inserted >= product_price:
                self.product_stock[product_key] -= 1
                self.total_inserted -= product_price
                self.change_to_return = self.total_inserted
                self.total_inserted = 0
                self.update_display()
                self.window['-STATUS-'].update('Please take your product and change.')
                if self.product_stock[product_key] == 0:
                    self.window[product_key].update('Sold Out', disabled=True)
                # Only activates the servo when a product is selected and paid for
                self.activate_servo()
            else:
                self.window['-STATUS-'].update('Insufficient funds. Please insert more coins.')
        else:
            self.window['-STATUS-'].update('Sold Out')
            
    def activate_servo(self):
        """Activates the servo to dispense the product."""
        if self.hardware_present:
            self.servo.max()
            sleep(1)
            self.servo.detach()

    def update_display(self):
        """Update the display of the Vending Machine."""
        self.window['-TOTAL-'].update(f'Total Inserted: ${self.total_inserted:.2f}')
        self.window['-CHANGE-'].update(f'Change to be Returned: ${self.change_to_return:.2f}')
        self.change_to_return = 0

# Coin values and product prices
coin_values = {
    '-NICKEL-': 0.05,
    '-DIME-': 0.10,
    '-QUARTER-': 0.25,
    '-LOONIE-': 1.00,
    '-TOONIE-': 2.00
}

product_prices = {
    '-Gum-': 0.50,
    '-Pop-': 0.75,
    '-Chips-': 1.00,
    '-Chocolate-': 1.50,
    '-Beer-': 2.00
}
# Define the layout of the GUI
layout = [
    [sg.Text('ENTER COINS', font='Helvetica 16', justification='center', pad=((0, 65), 0)),
     sg.Text('SELECT ITEM', font='Helvetica 16', justification='center', pad=((65, 0), 0))],
    [sg.Column([
        [sg.Button('5¢', size=(5, 2), key='-NICKEL-', button_color=('black', '#add8e6'))],
        [sg.Button('10¢', size=(5, 2), key='-DIME-', button_color=('black', '#add8e6'))],
        [sg.Button('25¢', size=(5, 2), key='-QUARTER-', button_color=('black', '#add8e6'))],
        [sg.Button('$1', size=(5, 2), key='-LOONIE-', button_color=('black', '#add8e6'))],
        [sg.Button('$2', size=(5, 2), key='-TOONIE-', button_color=('black', '#add8e6'))],
        [sg.Button('Return Coins', size=(10, 2), key='-RETURN-', button_color=('black', '#FF0000'))]
    ], element_justification='center', pad=((0, 75), 0)),
     sg.VerticalSeparator(),
     sg.Column([
         [sg.Button(f'Gum ($0.50)', size=(15, 2), key='-Gum-', button_color=('black', '#add8e6'))],
         [sg.Button(f'Pop ($0.75)', size=(15, 2), key='-Pop-', button_color=('black', '#add8e6'))],
         [sg.Button(f'Chips ($1.00)', size=(15, 2), key='-Chips-', button_color=('black', '#add8e6'))],
         [sg.Button(f'Chocolate ($1.50)', size=(15, 2), key='-Chocolate-', button_color=('black', '#add8e6'))],
         [sg.Button(f'Beer ($2.00)', size=(15, 2), key='-Beer-', button_color=('black', '#add8e6'))]
     ], element_justification='center', pad=((75, 0), 0))],
    [sg.Text('Total Inserted: $0.00', key='-TOTAL-', size=(30, 1))],
    [sg.Text('Change to be Returned: $0.00', key='-CHANGE-', size=(30, 1))],
    [sg.Text('Status: Ready', key='-STATUS-', size=(30, 1))]
]


# Create the window
window = sg.Window('Vending Machine GUI', layout, finalize=True)

# Create an instance of the VendingMachine class
vending_machine = VendingMachine(window)

# Event Loop to process "events" and get the "values" of the inputs
while True:
    event, values = window.read()

    if event == sg.WIN_CLOSED:  # if user closes window
        break
    elif event in coin_values:  # Coin insertion events
        vending_machine.add_coin(coin_values[event])
    elif event in product_prices:  # Product selection events
        vending_machine.select_product(event, product_prices[event])
    elif event == '-RETURN-':  # Return coins event
        vending_machine.return_coins()

# Clean up on exit
window.close()
print("Shutting down…")


