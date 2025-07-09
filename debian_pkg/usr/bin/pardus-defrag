#!/usr/bin/env python3

import sys
import subprocess
import os
import json
import re 

from PyQt5.QtWidgets import QApplication, QWidget, QVBoxLayout, QHBoxLayout, QLabel, QComboBox, QPushButton, QMessageBox, QProgressBar, QTextEdit
from PyQt5.QtCore import Qt, QThread, pyqtSignal

# Disk birleştirme işlemini ayrı bir thread'de çalıştırmak için
class DefragWorker(QThread):
    finished = pyqtSignal(str) 
    error = pyqtSignal(str) 
    progress = pyqtSignal(int) 

    def __init__(self, device_path):
        super().__init__()
        self.device_path = device_path

    def run(self):
        try:
            self.progress.emit(10) 

            command = ["pkexec", "/usr/sbin/e4defrag", self.device_path] 

            process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
            stdout, stderr = process.communicate() 

            if process.returncode == 0:
                self.progress.emit(100) 
                self.finished.emit(f"Disk birleştirme işlemi başarıyla tamamlandı: {self.device_path}\n\n{stdout}")
            else:
                self.error.emit(f"Disk birleştirme işlemi sırasında bir hata oluştu:\n\n{stderr}")

        except Exception as e:
            self.error.emit(f"Beklenmedik bir hata oluştu: {e}")

# e4defrag kontrolü için ayrı bir worker (Analiz Et butonu için)
class CheckDefragWorker(QThread):
    finished = pyqtSignal(int, str) # Skor ve orijinal stdout çıktısını da error kontrolü için gönderiyoruz
    error = pyqtSignal(str) 

    def __init__(self, device_path):
        super().__init__()
        self.device_path = device_path

    def run(self):
        try:
            command = ["pkexec", "/usr/sbin/e4defrag", "-c", self.device_path] 
            process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
            stdout, stderr = process.communicate()

            if process.returncode == 0:
                score = -1 
                lines = stdout.splitlines()

                found_score_line = False
                for line in lines:
                    if "Fragmentation score" in line:
                        match = re.search(r'Fragmentation score\s*(\d+)', line)
                        if match:
                            score = int(match.group(1))
                            found_score_line = True
                    if "No fragmentation found" in line and not found_score_line:
                        score = 0 

                self.finished.emit(score, stdout) 
            else:
                self.error.emit(f"Disk parçalanma kontrolü sırasında bir hata oluştu:\n{stderr}")

        except FileNotFoundError:
            self.error.emit("Hata: '/usr/sbin/e4defrag' komutu bulunamadı. Lütfen 'e2fsprogs' paketinin kurulu olduğundan emin olun.")
        except Exception as e:
            self.error.emit(f"Parçalanma kontrolü sırasında beklenmedik bir hata oluştu: {e}")


class DiskDefragmenterApp(QWidget):
    def __init__(self):
        super().__init__()
        self.initUI()
        self.worker = None 
        self.check_worker = None 

    def initUI(self):
        self.setWindowTitle('Pardus Disk Birleştirici')
        self.setGeometry(300, 300, 700, 450) 

        main_layout = QVBoxLayout()

        # Disk Seçim Kısmı
        disk_selection_layout = QHBoxLayout()
        disk_label = QLabel('Disk Seçin:')
        self.disk_combobox = QComboBox()

        disk_selection_layout.addWidget(disk_label)
        disk_selection_layout.addWidget(self.disk_combobox)
        main_layout.addLayout(disk_selection_layout)

        # Butonlar İçin Yatay Kutu
        button_layout = QHBoxLayout()

        # Analiz Et Butonu
        self.analyze_button = QPushButton('Analiz Et')
        self.analyze_button.clicked.connect(self.start_analysis)
        button_layout.addWidget(self.analyze_button)

        # Birleştir Butonu
        self.defrag_button = QPushButton('Birleştir')
        self.defrag_button.clicked.connect(self.start_defrag)
        self.defrag_button.setEnabled(False) 
        button_layout.addWidget(self.defrag_button)

        main_layout.addLayout(button_layout) # Yatay buton kutusunu ana düzene ekle

        # Bilgi Alanı (Disk bilgisi ve birleştirilebilirlik)
        self.info_label = QLabel("Seçilen disk hakkında bilgi burada görünecektir.")
        self.info_label.setWordWrap(True) 
        main_layout.addWidget(self.info_label)

        # Parçalanma Sonucu Alanı (Sadece puan ve yorum)
        self.defrag_result_label = QLabel("")
        self.defrag_result_label.setWordWrap(True)
        self.defrag_result_label.setStyleSheet("font-weight: bold; color: green;") 
        main_layout.addWidget(self.defrag_result_label)

        # İlerleme / Durum Mesaj Alanı (Mavi ve kalın yazı)
        self.status_message_label = QLabel("")
        self.status_message_label.setAlignment(Qt.AlignCenter)
        self.status_message_label.setStyleSheet("font-weight: bold; color: blue;") 
        self.status_message_label.hide() 
        main_layout.addWidget(self.status_message_label)

        # İlerleme Çubuğu (Şimdilik gizli kalacak)
        self.progress_bar = QProgressBar()
        self.progress_bar.setAlignment(Qt.AlignCenter)
        self.progress_bar.hide() 
        main_layout.addWidget(self.progress_bar)

        # Çıktı Alanı (Sadece işlem mesajları ve hatalar için)
        self.output_text_edit = QTextEdit()
        self.output_text_edit.setReadOnly(True)
        self.output_text_edit.setText("İşlem mesajları burada gösterilecektir.")
        main_layout.addWidget(self.output_text_edit)

        # Diski doldur ve ilk diski kontrol et (ama analiz yapma)
        self.populate_disks() 
        self.disk_combobox.currentIndexChanged.connect(self.on_disk_selection_changed) 
        self.on_disk_selection_changed() 

        self.setLayout(main_layout) 

    def populate_disks(self):
        try:
            command = ["lsblk", "-o", "NAME,FSTYPE,MOUNTPOINT,PATH", "--json"]
            result = subprocess.run(command, capture_output=True, text=True, check=True)
            disk_info = json.loads(result.stdout)

            self.disks = []
            self.disk_combobox.clear() 

            for device in disk_info.get("blockdevices", []):
                if "children" in device: 
                    for child_device in device["children"]:
                        self._add_disk_item(child_device)
                else: 
                    self._add_disk_item(device)

            if not self.disks:
                self.disk_combobox.addItem("Disk bulunamadı veya yetersiz yetki.")
                self.analyze_button.setEnabled(False)
                self.defrag_button.setEnabled(False)
                self.info_label.setText("Sistemde birleştirilebilecek EXT4 disk bulunamadı veya yetkisizlik.")
            else:
                self.analyze_button.setEnabled(True) 
                self.defrag_button.setEnabled(False) 
                self.on_disk_selection_changed()

        except FileNotFoundError:
            self.output_text_edit.setText("Hata: 'lsblk' komutu bulunamadı. Lütfen 'util-linux' paketinin kurulu olduğundan emin olun.")
            self.analyze_button.setEnabled(False)
            self.defrag_button.setEnabled(False)
        except Exception as e:
            self.output_text_edit.setText(f"Diskler listelenirken beklenmedik bir hata oluştu: {e}")
            self.analyze_button.setEnabled(False)
            self.defrag_button.setEnabled(False)

    def _add_disk_item(self, device):
        fstype = device.get("fstype")
        mountpoint = device.get("mountpoint")
        path = device.get("path") 

        if fstype and path and path.startswith("/dev/"):
            display_name = f"{path} ({fstype})"
            if mountpoint:
                display_name += f" - {mountpoint}"

            self.disks.append({
                "path": path,
                "fstype": fstype,
                "mountpoint": mountpoint if mountpoint else "Yok"
            })
            self.disk_combobox.addItem(display_name)

    def on_disk_selection_changed(self):
        selected_index = self.disk_combobox.currentIndex()
        self.defrag_result_label.clear() # Yeni disk seçildiğinde parçalanma sonucunu temizle

        if selected_index == -1 or not self.disks or selected_index >= len(self.disks):
            self.info_label.setText("Lütfen geçerli bir disk seçin.")
            self.analyze_button.setEnabled(False)
            self.defrag_button.setEnabled(False)
            self.status_message_label.hide()
            return

        selected_disk_info = self.disks[selected_index]
        fstype = selected_disk_info["fstype"]
        path = selected_disk_info["path"]
        mountpoint = selected_disk_info["mountpoint"]

        if fstype == "ext4":
            self.info_label.setText(
                f"Seçilen disk: <b>{path}</b><br>"
                f"Dosya Sistemi: <b>{fstype}</b><br>"
                f"Bağlama Noktası: <b>{mountpoint}</b><br><br>"
                f"<b>Bu disk birleştirilebilir türden.</b> Analiz etmek için 'Analiz Et' butonuna tıklayın." # Burayı güncelledik
            )
            self.analyze_button.setEnabled(True) 
            self.defrag_button.setEnabled(False) 
        else:
            self.info_label.setText(
                f"Seçilen disk: <b>{path}</b><br>"
                f"Dosya Sistemi: <b>{fstype}</b><br>"
                f"Bağlama Noktası: <b>{mountpoint}</b><br><br>"
                f"<span style='color:red;'><b>Uyarı:</b> Linux ortamında bu diske doğrudan disk birleştirme işlemi yapılamaz.</span>"
            )
            self.analyze_button.setEnabled(False) 
            self.defrag_button.setEnabled(False)

        self.status_message_label.hide() 
        self.output_text_edit.clear() 

    def start_analysis(self):
        selected_index = self.disk_combobox.currentIndex()
        if selected_index == -1 or not self.disks or selected_index >= len(self.disks):
            QMessageBox.warning(self, "Uyarı", "Lütfen bir disk seçin.")
            return

        selected_disk_info = self.disks[selected_index]
        fstype = selected_disk_info["fstype"]
        path = selected_disk_info["path"]

        if fstype != "ext4":
            QMessageBox.warning(self, "Hata", f"Seçilen disk ({path}) EXT4 formatında değil. Sadece EXT4 diskler birleştirilebilir.")
            return

        # Analiz süresince butonları pasif yap
        self.analyze_button.setEnabled(False)
        self.defrag_button.setEnabled(False)
        self.disk_combobox.setEnabled(False)

        self.status_message_label.setText("Parçalanma derecesi kontrol ediliyor... (Şifre sorabilir)")
        self.status_message_label.show()
        self.output_text_edit.clear()
        self.output_text_edit.append("Disk analiz ediliyor...\n")
        self.defrag_result_label.clear() # Yeni analiz öncesi sonucu temizle

        self.check_worker = CheckDefragWorker(path)
        self.check_worker.finished.connect(
            lambda score, full_output: self.display_defrag_score(path, fstype, selected_disk_info["mountpoint"], score, full_output)
        )
        self.check_worker.error.connect(self.display_defrag_check_error)
        self.check_worker.start()

    def display_defrag_score(self, path, fstype, mountpoint, score, full_output): 
        self.analyze_button.setEnabled(True) 
        self.disk_combobox.setEnabled(True) 
        self.status_message_label.hide() 
        self.defrag_button.setEnabled(True) 

        # INFO LABEL SADECE DİSK BİLGİSİ VE BİRLEŞTİRİLEBİLİRLİK İÇİN
        current_info_text = (
            f"Seçilen disk: <b>{path}</b><br>"
            f"Dosya Sistemi: <b>{fstype}</b><br>"
            f"Bağlama Noktası: <b>{mountpoint}</b><br><br>"
            f"<b>Bu disk birleştirilebilir türden.</b>" # Analiz sonrası da bu bilgi kalacak
        )
        self.info_label.setText(current_info_text)

        # PARÇALANMA SONUCU İÇİN AYRI ALAN (defrag_result_label)
        result_text = ""
        if score == 0:
            result_text = "Parçalanma Puanı: <b>0</b>. Disk parçalanmış değil. Birleştirmeye gerek yok."
            self.defrag_result_label.setStyleSheet("font-weight: bold; color: green;")
        elif 1 <= score <= 30:
            result_text = f"Parçalanma Puanı: <b>{score}</b>. Çok az miktarda parçalanma var. Birleştirmeye genellikle gerek yoktur."
            self.defrag_result_label.setStyleSheet("font-weight: bold; color: blue;")
        elif 31 <= score <= 55:
            result_text = f"Parçalanma Puanı: <b>{score}</b>. Orta derecede parçalanmış. Birleştirme önerilir."
            self.defrag_result_label.setStyleSheet("font-weight: bold; color: orange;")
        elif score >= 56:
            result_text = f"Parçalanma Puanı: <b>{score}</b>. Yüksek derecede parçalanma var! Disk birleştirmeye ihtiyaç duyuyor."
            self.defrag_result_label.setStyleSheet("font-weight: bold; color: red;")
        else: 
            result_text = "Parçalanma Puanı belirlenemedi veya bilinmeyen bir durum oluştu."
            self.defrag_result_label.setStyleSheet("font-weight: bold; color: gray;")
            self.defrag_button.setEnabled(False) 

        self.defrag_result_label.setText(result_text)
        self.output_text_edit.append("Disk analizi tamamlandı.")

    def display_defrag_check_error(self, message):
        # Hata durumunda info_label'ı güncelle
        selected_index = self.disk_combobox.currentIndex()
        selected_disk_info = self.disks[selected_index]
        path = selected_disk_info["path"]
        fstype = selected_disk_info["fstype"]
        mountpoint = selected_disk_info["mountpoint"]

        self.info_label.setText(
            f"Seçilen disk: <b>{path}</b><br>"
            f"Dosya Sistemi: <b>{fstype}</b><br>"
            f"Bağlama Noktası: <b>{mountpoint}</b><br><br>"
            f"<span style='color:red;'><b>Parçalanma kontrolü sırasında bir hata oluştu: {message}</b></span>" # Hata mesajını buraya taşıdık
        )
        self.analyze_button.setEnabled(True) 
        self.defrag_button.setEnabled(False)
        self.disk_combobox.setEnabled(True)
        self.status_message_label.hide()
        self.output_text_edit.append(message) # output_text_edit'te de kalsın
        self.defrag_result_label.clear()


    def start_defrag(self):
        selected_index = self.disk_combobox.currentIndex()
        if selected_index == -1 or not self.disks or selected_index >= len(self.disks):
            QMessageBox.warning(self, "Uyarı", "Lütfen bir disk seçin.")
            return

        selected_disk_info = self.disks[selected_index]
        device_path = selected_disk_info["path"]
        fstype = selected_disk_info["fstype"]

        if fstype != "ext4":
            QMessageBox.warning(self, "Hata", f"Seçilen disk ({device_path}) EXT4 formatında değil. Sadece EXT4 diskler birleştirilebilir.")
            return

        reply = QMessageBox.question(self, 'Onay', 
                                     f'"{device_path}" üzerindeki disk birleştirme işlemini başlatmak istediğinizden emin misiniz? Bu işlem biraz zaman alabilir ve sistem kaynaklarını kullanabilir.',
                                     QMessageBox.Yes | QMessageBox.No, QMessageBox.No) 

        if reply == QMessageBox.Yes:
            self.output_text_edit.clear()
            self.output_text_edit.append(f"Disk birleştirme işlemi başlatılıyor: <b>{device_path}</b>\n")

            self.defrag_button.setEnabled(False)
            self.analyze_button.setEnabled(False)
            self.disk_combobox.setEnabled(False)

            self.progress_bar.hide() 
            self.status_message_label.setText("İşlem sürüyor, lütfen bekleyin. Bu işlem uzun sürebilir... (Şifre sorabilir)")
            self.status_message_label.show()

            self.worker = DefragWorker(device_path)
            self.worker.finished.connect(self.defrag_finished)
            self.worker.error.connect(self.defrag_error)
            self.worker.progress.connect(self.progress_bar.setValue) 
            self.worker.start() 

    def defrag_finished(self, message):
        self.output_text_edit.append(message)
        QMessageBox.information(self, "Başarılı", "Disk birleştirme işlemi tamamlandı.")
        self.reset_ui()

    def defrag_error(self, message):
        self.output_text_edit.append(message)
        QMessageBox.critical(self, "Hata", "Disk birleştirme işlemi sırasında bir hata oluştu.")
        self.reset_ui()

    def closeEvent(self, event):
        if (self.worker and self.worker.isRunning()) or (self.check_worker and self.check_worker.isRunning()):
            reply = QMessageBox.question(self, 'Uyarı',
                                         "Devam eden bir işlem var (analiz veya birleştirme). Uygulamayı şimdi kapatmak veri kaybına neden olabilir veya işlemi kesintiye uğratabilir. Yine de kapatmak istiyor musunuz?",
                                         QMessageBox.Yes | QMessageBox.No, QMessageBox.No)
            if reply == QMessageBox.Yes:
                if self.worker and self.worker.isRunning():
                    self.worker.terminate() 
                    self.worker.wait(5000) 
                if self.check_worker and self.check_worker.isRunning():
                    self.check_worker.terminate() 
                    self.check_worker.wait(5000) 
                event.accept()
            else:
                event.ignore()
        else:
            event.accept()

    def reset_ui(self):
        self.defrag_button.setEnabled(False) 
        self.analyze_button.setEnabled(True) 
        self.disk_combobox.setEnabled(True)
        self.progress_bar.hide()
        self.progress_bar.setValue(0)
        self.status_message_label.hide() 
        self.worker = None 
        self.check_worker = None
        self.on_disk_selection_changed() 

if __name__ == '__main__':
    app = QApplication(sys.argv)
    ex = DiskDefragmenterApp()
    ex.show()
    sys.exit(app.exec_())
