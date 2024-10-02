# Image-Dimension-Checker
This is an application that check dimensions file type and some other information of image 
import os
from tkinter import Tk, Label, Button, filedialog, StringVar, Frame
from PIL import Image, ExifTags, ImageDraw, ImageFont
import platform

def get_desktop_path():
    """Get the path to the user's desktop."""
    if platform.system() == "Windows":
        return os.path.join(os.path.join(os.environ['USERPROFILE']), 'Desktop')
    elif platform.system() == "Darwin":  # macOS
        return os.path.join(os.path.expanduser('~'), 'Desktop')
    else:  # Linux
        return os.path.join(os.path.expanduser('~'), 'Desktop')

def select_image():
    """Open a file dialog to select an image and display its properties."""
    image_path = filedialog.askopenfilename(filetypes=[("Image files", "*.jpg;*.jpeg;*.png;*.bmp;*.gif")])
    
    if not image_path:
        return
    
    # Check image properties
    properties = check_image_properties(image_path)
    
    # Generate reports and images
    generate_html_report(image_path, *properties)
    create_properties_image(image_path, properties)
    save_properties_to_text(image_path, properties)
    
    # Display properties in GUI
    display_properties(image_path, properties)

def check_image_properties(image_path):
    """Get the properties of the image."""
    with Image.open(image_path) as img:
        width, height = img.size
        mode = img.mode
        
        # Calculate bit depth based on the image mode
        bit_depth = {
            "1": 1,
            "L": 8,
            "P": 8,
            "RGB": 24,
            "RGBA": 32,
            "CMYK": 32
        }.get(mode, "Unknown")

        dpi = img.info.get('dpi', (72, 72))
        file_format = img.format
        transparency = "Yes" if mode in ("RGBA", "LA") else "No"
        exif_data = {}
        
        if hasattr(img, '_getexif'):
            exif = img._getexif()
            if exif:
                exif_data = {ExifTags.TAGS.get(tag): value for tag, value in exif.items() if tag in ExifTags.TAGS}

    # Get file size in bytes
    file_size = os.path.getsize(image_path)

    return (width, height, file_size, mode, bit_depth, dpi, file_format, transparency, exif_data)

def format_file_size(file_size):
    """Convert file size to KB or MB."""
    if file_size < 1024:
        return f"{file_size} Bytes"
    elif file_size < 1024**2:
        return f"{file_size / 1024:.2f} KB"
    else:
        return f"{file_size / 1024**2:.2f} MB"

def display_properties(image_path, properties):
    """Display the properties of the selected image in the GUI."""
    width, height, file_size, mode, bit_depth, dpi, file_format, transparency, exif_data = properties
    formatted_file_size = format_file_size(file_size)

    properties_text = (
        f"Image Path: {image_path}\n"
        f"Width: {width}px\n"
        f"Height: {height}px\n"
        f"File Size: {formatted_file_size}\n"
        f"Mode: {mode}\n"
        f"Bit Depth: {bit_depth}\n"
        f"DPI: {dpi[0]} x {dpi[1]}\n"
        f"File Format: {file_format}\n"
        f"Transparency: {transparency}\n"
        f"EXIF Data: {exif_data if exif_data else 'No EXIF Data'}"
    )
    
    properties_var.set(properties_text)

def generate_html_report(image_path, width, height, file_size, mode, bit_depth, dpi, file_format, transparency, exif_data):
    """Generate an HTML report of the image properties."""
    aspect_ratio = f"{width}/{height}"
    formatted_file_size = format_file_size(file_size)

    # Create HTML content
    html_content = f"""
    <html>
    <head>
        <title>Image Report</title>
        <style>
            body {{ font-family: Arial, sans-serif; }}
            table {{ width: 50%; margin: 20px auto; border-collapse: collapse; }}
            th, td {{ padding: 10px; border: 1px solid #ccc; }}
            th {{ background-color: #4CAF50; color: white; }}
            h2 {{ text-align: center; }}
        </style>
    </head>
    <body>
        <h2>Image Path</h2>
        <p style="text-align: center;">{image_path}</p>
        <h2>Image Preview</h2>
        <img src="file:///{image_path}" alt="Image" style="display: block; margin: 0 auto; max-width: 80%; height: auto;"/>
        
        <h2>General Image Data</h2>
        <table>
            <tr>
                <th>File Size</th>
                <td>{formatted_file_size}</td>
            </tr>
            <tr>
                <th>Width in px</th>
                <td>{width}</td>
            </tr>
            <tr>
                <th>Height in px</th>
                <td>{height}</td>
            </tr>
            <tr>
                <th>Aspect Ratio</th>
                <td>{aspect_ratio}</td>
            </tr>
            <tr>
                <th>DPI</th>
                <td>{dpi[0]} x {dpi[1]}</td>
            </tr>
            <tr>
                <th>File Format</th>
                <td>{file_format}</td>
            </tr>
            <tr>
                <th>Transparency</th>
                <td>{transparency}</td>
            </tr>
        </table>

        <h2>Image Metadata</h2>
        <table>
            <tr>
                <th>Tag Name</th>
                <th>Tag Description</th>
            </tr>
            <tr>
                <td>Image Width</td>
                <td>{width}px</td>
            </tr>
            <tr>
                <td>Image Height</td>
                <td>{height}px</td>
            </tr>
            <tr>
                <td>Bit Depth</td>
                <td>{bit_depth}</td>
            </tr>
            <tr>
                <td>Color Type</td>
                <td>{mode}</td>
            </tr>
            <tr>
                <td>EXIF Data</td>
                <td>{exif_data if exif_data else "No EXIF Data"}</td>
            </tr>
        </table>
    </body>
    </html>
    """
    
    # Save the HTML content to a file on the desktop
    desktop_path = get_desktop_path()
    html_file_path = os.path.join(desktop_path, "image_report.html")
    with open(html_file_path, "w") as file:
        file.write(html_content)

    print(f"HTML report generated: {html_file_path}")

def create_properties_image(image_path, properties):
    """Create an image showing the properties of the selected image."""
    width, height, file_size, mode, bit_depth, dpi, file_format, transparency, exif_data = properties
    formatted_file_size = format_file_size(file_size)

    # Create a blank image
    img = Image.new('RGB', (800, 600), (255, 255, 255))
    draw = ImageDraw.Draw(img)
    
    # Optionally, you can use a font, if you have a .ttf file available
    font = ImageFont.load_default()

    # Draw text on the image
    text_lines = [
        f"Image Path: {image_path}",
        f"Width: {width}px",
        f"Height: {height}px",
        f"File Size: {formatted_file_size}",
        f"Mode: {mode}",
        f"Bit Depth: {bit_depth}",
        f"DPI: {dpi[0]} x {dpi[1]}",
        f"File Format: {file_format}",
        f"Transparency: {transparency}",
        f"EXIF Data: {exif_data if exif_data else 'No EXIF Data'}"
    ]

    y_text = 10
    for line in text_lines:
        draw.text((10, y_text), line, fill=(0, 0, 0), font=font)
        y_text += 20

    # Save the properties image on the desktop
    desktop_path = get_desktop_path()
    properties_image_path = os.path.join(desktop_path, "image_properties.png")
    img.save(properties_image_path)

    print(f"Properties image generated: {properties_image_path}")

def save_properties_to_text(image_path, properties):
    """Save the properties of the image to a text file."""
    width, height, file_size, mode, bit_depth, dpi, file_format, transparency, exif_data = properties
    formatted_file_size = format_file_size(file_size)

    # Create text content
    text_content = f"""
    Image Path: {image_path}
    Width: {width}px
    Height: {height}px
    File Size: {formatted_file_size}
    Mode: {mode}
    Bit Depth: {bit_depth}
    DPI: {dpi[0]} x {dpi[1]}
    File Format: {file_format}
    Transparency: {transparency}
    EXIF Data: {exif_data if exif_data else 'No EXIF Data'}
    """

    # Save the text content to a file on the desktop
    desktop_path = get_desktop_path()
    text_file_path = os.path.join(desktop_path, "image_properties.txt")
    with open(text_file_path, "w") as file:
        file.write(text_content)

    print(f"Text file generated: {text_file_path}")

# Create the main application window
root = Tk()
root.title("Image Size Finder")
root.geometry("400x500")
root.configure(bg='#f0f0f0')  # Set background color

# Create a frame for the interface and center it
frame = Frame(root, bg='#4CAF50')  # Frame color
frame.pack(expand=True)

Label(frame, text="Select an image to get its properties:", font=("Arial", 14), bg='#4CAF50', fg='white').pack()

Button(frame, text="Select Image", command=select_image, font=("Arial", 12), bg='#ffffff', fg='#4CAF50').pack(pady=20)

properties_var = StringVar()

Label(frame, textvariable=properties_var, font=("Arial", 10), justify='left', bg='#4CAF50', fg='white').pack(pady=20)

# Start the application
root.mainloop()
