import tkinter as tk
from tkinter import messagebox
import datetime
import os
import qrcode
import base64
import xml.etree.ElementTree as ET
import webbrowser
import random
from PIL import Image

def submit():
    year = year_entry.get()
    make = make_entry.get()
    model = model_entry.get()
    vin = vin_entry.get()

    # Random selections
    insured_by = random.choice(insurers)
    policy_number = random.randint(1000, 999999)
    issuing_authority = random.randint(1000, 9999)

    # Dates and permit number calculation
    current_date = datetime.datetime.now()
    issue_date = current_date.strftime("%d %b %Y").upper()
    effective_date = issue_date
    expiration_date = (current_date + datetime.timedelta(days=21)).strftime("%d %b %Y").upper()
    prepermnum = int(940000 + (((datetime.datetime.now() - datetime.datetime(2024, 1, 1)).total_seconds() / 60) * 0.05))
    permnum = "{:07d}".format(prepermnum)

    data = f"{effective_date}/{expiration_date}\n{permnum}\n{vin}\n{issuing_authority}"
    qr_img_path = generate_qr_code(data)
    rq_img_path = generate_rq_code(data)
    update_svg_and_save(year, make, model, vin, permnum, issue_date, effective_date, expiration_date, insured_by, policy_number, issuing_authority, qr_img_path, rq_img_path)

def generate_qr_code(data):
    qr = qrcode.QRCode(
        version=1,
        error_correction=qrcode.constants.ERROR_CORRECT_H,
        box_size=10,
        border=4,
    )
    qr.add_data(data)
    qr.make(fit=True)
    img = qr.make_image(fill_color="black", back_color="white")
    img_path = os.path.join(os.getcwd(), 'qrcode.png')
    img.save(img_path)
    return img_path

def generate_rq_code(data):
    rq = qrcode.QRCode(
        version=1,
        error_correction=qrcode.constants.ERROR_CORRECT_H,
        box_size=5,
        border=2,
    )
    rq.add_data(data)
    rq.make(fit=True)
    alt = rq.make_image(fill_color="black", back_color="white")
    alt_path = os.path.join(os.getcwd(), 'rqcode.png' )
    alt.save(alt_path)

    return alt_path


def update_svg_and_save(year, make, model, vin, permnum, issue_date, effective_date, expiration_date, insured_by, policy_number, issuing_authority, qr_img_path, rq_img_path):
    template_path = 'template.svg'  # Specify the correct path to your SVG template
    tree = ET.parse(template_path)
    root = tree.getroot()

    fields = {
        "tspan34": year,
        "tspan35": make,
        "tspan36": model,
        "tspan37": vin,
        "tspan43": str(permnum),
        "tspan78": issue_date,
        "tspan66": effective_date,
        "tspan25": expiration_date,
        "tspan64": insured_by,
        "tspan80": str(policy_number),
        "tspan63": str(issuing_authority),
    }

    for id, value in fields.items():
        element = root.find(f".//*[@id='{id}']")
        if element is not None:
            element.text = value

    # Embed the QR code
    with open(qr_img_path, "rb") as image_file:
        encoded_string = base64.b64encode(image_file.read()).decode('utf-8')
    qr_placeholder = root.find(".//*[@id='image1-5']")  # Adjust this as necessary
    if qr_placeholder is not None:
        qr_placeholder.set('{http://www.w3.org/1999/xlink}href', f'data:image/png;base64,{encoded_string}')

    with open(rq_img_path, "rb") as image_file:
        encoded_string = base64.b64encode(image_file.read()).decode('utf-8')
    rq_placeholder = root.find(".//*[@id='image1-05']")
    if rq_placeholder is not None:
        rq_placeholder.set('{http://www.w3.org/1999/xlink}href', f'data:image/png;base64, {encoded_string}')

        output_svg_path = 'updated_permit.svg'
    tree.write(output_svg_path)

    messagebox.showinfo("Success", f"Updated permit saved as {output_svg_path}")
    webbrowser.open('file://' + os.path.realpath(output_svg_path))

# Insurance providers
insurers = [
    "PROGRESSIVE", "GEICO", "ALLSTATE", "STATE FARM", "FARMERS",
    "THEGENERAL.COM", "VERN FONK", "AMERICAN FINANCIAL",
    "COUNTRY FINANCIAL", "LIBERTY MUTUAL", "MERCURY", "TOGGLE"
]

# Setup the GUI
app = tk.Tk()
app.title("Temporary Permit Generator")

labels = ['Year', 'Make', 'Model', 'VIN']
entries = {}
row = 0
for label in labels:
    tk.Label(app, text=label).grid(row=row, column=0)
    entry = tk.Entry(app)
    entry.grid(row=row, column=1)
    entries[label] = entry
    row += 1

year_entry, make_entry, model_entry, vin_entry = entries['Year'], entries['Make'], entries['Model'], entries['VIN']

submit_btn = tk.Button(app, text="Create OR Temporary Permit", command=submit)
submit_btn.grid(row=row, column=0, columnspan=2)

app.mainloop()
