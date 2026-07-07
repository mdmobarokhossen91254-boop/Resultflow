# Resultflow

import tkinter as tk
from tkinter import filedialog, messagebox
import PyPDF2
import re
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib import colors

class ResultAnalyzerApp:
    def __init__(self, root):
        self.root = root
        self.root.title("দিনাজপুর পলিটেকনিক রেজাল্ট অ্যানালাইজার")
        self.root.geometry("800x600")

        self.pdf_path = ""
        self.extracted_data = []
        self.filtered_data = []

        self.create_widgets()

    def create_widgets(self):
        # PDF File Selection
        file_frame = tk.LabelFrame(self.root, text="পিডিএফ ফাইল নির্বাচন করুন", padx=10, pady=10)
        file_frame.pack(pady=10, padx=10, fill="x")

        self.file_label = tk.Label(file_frame, text="কোনো ফাইল নির্বাচন করা হয়নি।")
        self.file_label.pack(side="left", fill="x", expand=True)

        browse_button = tk.Button(file_frame, text="ব্রাউজ করুন", command=self.browse_pdf)
        browse_button.pack(side="right")

        # Department and Roll Range Info
        info_frame = tk.LabelFrame(self.root, text="বিভাগ ও রোল নম্বর তথ্য", padx=10, pady=10)
        info_frame.pack(pady=10, padx=10, fill="x")

        tk.Label(info_frame, text="ডিপার্টমেন্ট: সকল  টেকনোলজি").pack(anchor="w")
        tk.Label(info_frame, text="গ্রুপ: সকল ").pack(anchor="w")
        tk.Label(info_frame, text="শিফট: উভয় ").pack(anchor="w")
        tk.Label(info_frame, text="রোল রেঞ্জ: 200013 - 885745").pack(anchor="w")

        # Actions
        action_frame = tk.LabelFrame(self.root, text="অ্যাকশন", padx=10, pady=10)
        action_frame.pack(pady=10, padx=10, fill="x")

        process_button = tk.Button(action_frame, text="রেজাল্ট প্রসেস করুন ও রিপোর্ট তৈরি করুন", command=self.process_and_generate_report)
        process_button.pack(pady=5)

        # Search by Roll
        search_frame = tk.LabelFrame(self.root, text="রোল নম্বর দিয়ে সার্চ করুন", padx=10, pady=10)
        search_frame.pack(pady=10, padx=10, fill="x")

        tk.Label(search_frame, text="রোল নম্বর:").pack(side="left")
        self.search_roll_entry = tk.Entry(search_frame, width=20)
        self.search_roll_entry.pack(side="left", padx=5)

        search_button = tk.Button(search_frame, text="সার্চ করুন", command=self.search_roll)
        search_button.pack(side="left")

        self.search_result_label = tk.Label(search_frame, text="")
        self.search_result_label.pack(pady=5)

        # Output Log
        self.log_text = tk.Text(self.root, height=10, state="disabled")
        self.log_text.pack(pady=10, padx=10, fill="both", expand=True)

    def log_message(self, message):
        self.log_text.config(state="normal")
        self.log_text.insert(tk.END, message + "\n")
        self.log_text.see(tk.END)
        self.log_text.config(state="disabled")

    def browse_pdf(self):
        filetypes = [("PDF files", "*.pdf")]
        path = filedialog.askopenfilename(filetypes=filetypes)
        if path:
            self.pdf_path = path
            self.file_label.config(text=f"নির্বাচিত ফাইল: {self.pdf_path}")
            self.log_message(f"পিডিএফ ফাইল নির্বাচন করা হয়েছে: {self.pdf_path}")

    def extract_data_from_pdf(self):
        if not self.pdf_path:
            messagebox.showerror("ত্রুটি", "অনুগ্রহ করে একটি পিডিএফ ফাইল নির্বাচন করুন।")
            return False

        self.log_message("পিডিএফ থেকে ডাটা এক্সট্রাক্ট করা হচ্ছে...")
        try:
            reader = PyPDF2.PdfReader(self.pdf_path)
            data = []
            # Assuming Dinajpur Polytechnic starts around page 22 based on previous observation
            # And assuming the target department/group is within a continuous block of pages
            # For a more robust solution, one would need to parse department headers more explicitly
            
            # We will iterate through a reasonable range of pages where Dinajpur Polytechnic results might be
            # and then filter by roll number.
            # A more advanced parser would identify 'Dinajpur Polytechnic Institute' and then 'Computer Science and Technology'
            # and then 'Group A' to narrow down the pages. For this specific request, the roll range is key.

            # Let's assume Dinajpur Polytechnic data is from page 22 onwards for a few hundred pages.
            # We will extract all rolls and then filter.
            
            # The previous observation showed Dinajpur Polytechnic starting on page 22.
            # Let's extract from page 21 (0-indexed) up to a reasonable number of pages, say 200 pages after that.
            # This is a heuristic; a more precise method would involve dynamic page range detection.
            start_page_index = 21 # Page 22 in PDF
            end_page_index = min(len(reader.pages), start_page_index + 200) # Limit to 200 pages or end of doc

            for i in range(start_page_index, end_page_index):
                text = reader.pages[i].extract_text()
                
                # Extract passed students (Roll and GPA/CGPA)
                passed = re.findall(r'(\d{6})\s*\(\s*([\d.]+)\s*\)', text)
                for roll, gpa in passed:
                    data.append({'roll': roll, 'result': float(gpa), 'status': 'Passed'})
                
                # Extract failed students (Roll and Subject Codes)
                # This regex looks for a 6-digit roll followed by { and then anything until }.
                # The content inside {} is treated as subject codes.
                failed = re.findall(r'(\d{6})\s*\{([^\}]+)\}', text)
                for roll, subjects in failed:
                    data.append({'roll': roll, 'result': subjects.strip(), 'status': 'Referred'})
            
            self.extracted_data = data
            self.log_message(f"মোট {len(self.extracted_data)} টি রেকর্ড এক্সট্রাক্ট করা হয়েছে।")
            return True
        except Exception as e:
            messagebox.showerror("ত্রুটি", f"পিডিএফ থেকে ডাটা এক্সট্রাক্ট করতে ব্যর্থ: {e}")
            self.log_message(f"ত্রুটি: পিডিএফ থেকে ডাটা এক্সট্রাক্ট করতে ব্যর্থ: {e}")
            return False

    def filter_and_rank_data(self):
        self.log_message("ডাটা ফিল্টার করা হচ্ছে (রোল রেঞ্জ: 200013-885745)...")
        target_min_roll = 200013
        target_max_roll = 885745

        filtered = []
        for student in self.extracted_data:
            try:
                roll = int(student['roll'])
                if target_min_roll <= roll <= target_max_roll:
                    filtered.append(student)
            except ValueError:
                # Handle cases where roll might not be a valid integer
                continue
        
        # Sort by CGPA (descending) for ranking. Referred students will be at the bottom.
        # Assuming 'result' is float for 'Passed' students and string for 'Referred'.
        # We'll put 'Referred' students at the end for ranking purposes.
        self.filtered_data = sorted(filtered, key=lambda x: (x['status'] != 'Passed', -x['result'] if x['status'] == 'Passed' else 0))
        
        self.log_message(f"ফিল্টার করা হয়েছে {len(self.filtered_data)} টি রেকর্ড।")

    def generate_pdf_report(self):
        if not self.filtered_data:
            messagebox.showinfo("তথ্য", "রিপোর্ট তৈরি করার জন্য কোনো ডাটা নেই। প্রথমে ডাটা প্রসেস করুন।")
            return

        report_filename = "dinajpur_poly_cst_results_report.pdf"
        self.log_message(f"পিডিএফ রিপোর্ট তৈরি করা হচ্ছে: {report_filename}")

        doc = SimpleDocTemplate(report_filename, pagesize=letter)
        styles = getSampleStyleSheet()
        story = []

        # Title
        story.append(Paragraph("দিনাজপুর পলিটেকনিক ইনস্টিটিউট", styles['h1']))
        story.append(Paragraph("কম্পিউটার সায়েন্স এন্ড টেকনোলজি ডিপার্টমেন্ট (এ গ্রুপ, ১ম শিফট)", styles['h2']))
        story.append(Paragraph("রোল নম্বর: 302080 - 302131", styles['h3']))
        story.append(Spacer(1, 0.2 * 2.54 * 72)) # 0.2 inch spacer

        # Table Data
        table_data = [['র‍্যাঙ্ক', 'রোল নম্বর', 'ফলাফল (CGPA/বিষয় কোড)', 'স্ট্যাটাস']]
        rank = 1
        for student in self.filtered_data:
            if student['status'] == 'Passed':
                table_data.append([str(rank), student['roll'], f"{student['result']:.2f}", student['status']])
                rank += 1
            else:
                table_data.append(['-', student['roll'], student['result'], student['status']])

        table_style = TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.HexColor('#4CAF50')),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
            ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
            ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
            ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
            ('BACKGROUND', (0, 1), (-1, -1), colors.beige),
            ('GRID', (0, 0), (-1, -1), 1, colors.black),
            ('BOX', (0, 0), (-1, -1), 1, colors.black),
            ('LEFTPADDING', (0,0), (-1,-1), 6),
            ('RIGHTPADDING', (0,0), (-1,-1), 6),
            ('TOPPADDING', (0,0), (-1,-1), 6),
            ('BOTTOMPADDING', (0,0), (-1,-1), 6),
        ])

        # Add alternating row colors for better readability
        for i in range(1, len(table_data)):
            if i % 2 == 0:
                table_style.add('BACKGROUND', (0, i), (-1, i), colors.HexColor('#E0E0E0'))
            else:
                table_style.add('BACKGROUND', (0, i), (-1, i), colors.HexColor('#F5F5F5'))

        table = Table(table_data)
        table.setStyle(table_style)
        story.append(table)
        story.append(Spacer(1, 0.2 * 2.54 * 72))
        story.append(Paragraph("এই রিপোর্টটি M দ্বারা তৈরি করা হয়েছে।", styles['Normal']))

        try:
            doc.build(story)
            messagebox.showinfo("সফল", f"পিডিএফ রিপোর্ট সফলভাবে তৈরি হয়েছে: {report_filename}")
            self.log_message(f"পিডিএফ রিপোর্ট সফলভাবে তৈরি হয়েছে: {report_filename}")
        except Exception as e:
            messagebox.showerror("ত্রুটি", f"পিডিএফ রিপোর্ট তৈরি করতে ব্যর্থ: {e}")
            self.log_message(f"ত্রুটি: পিডিএফ রিপোর্ট তৈরি করতে ব্যর্থ: {e}")

    def process_and_generate_report(self):
        if self.extract_data_from_pdf():
            self.filter_and_rank_data()
            self.generate_pdf_report()

    def search_roll(self):
        search_roll_num = self.search_roll_entry.get().strip()
        if not search_roll_num:
            self.search_result_label.config(text="অনুগ্রহ করে একটি রোল নম্বর লিখুন।", fg="red")
            return

        found = False
        for student in self.filtered_data:
            if student['roll'] == search_roll_num:
                rank = "-"
                if student['status'] == 'Passed':
                    # Find rank for passed students
                    passed_students = [s for s in self.filtered_data if s['status'] == 'Passed']
                    for i, s in enumerate(passed_students):
                        if s['roll'] == search_roll_num:
                            rank = str(i + 1)
                            break

                result_text = f"{student['result']:.2f}" if student['status'] == 'Passed' else student['result']
                self.search_result_label.config(text=f"রোল: {student['roll']}, ফলাফল: {result_text}, স্ট্যাটাস: {student['status']}, র‍্যাঙ্ক: {rank}", fg="blue")
                found = True
                break
        
        if not found:
            self.search_result_label.config(text=f"রোল নম্বর {search_roll_num} পাওয়া যায়নি।", fg="red")

if __name__ == "__main__":
    root = tk.Tk()
    app = ResultAnalyzerApp(root)
    root.mainloop()

