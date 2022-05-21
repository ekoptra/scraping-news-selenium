# Scraping News with Selenium

Kode untuk melakukan scraping pada website `detik`, `republika`, dan `kompas`. Untuk setiap website perlu dikumpulkan link artikelnya terlebih dahulu. Untuk `detik` sudah disertai dengan scraping link berita berdasarkan keyword tertentu. Tetapi untuk `republika` dan `kompas` belum bisa dilakukan karena hasil pencarian keyword untuk website tersebut dilakukan oleh [Programmable Search Engine - Google](https://programmablesearchengine.google.com/about/) dan sulit untuk membuat scrapernya

Scraping dilakukan menggunakan [Selenium](https://www.selenium.dev/) sehingga perlu dilakukan configurasi terlebih dahulu yaitu install library `pip install selenium`, kemudian download [webdriver Chrome](https://chromedriver.chromium.org/downloads) yang versinya sesuai dengan Google Chrome yang digunakan dan atur `PATH` dari webdrivernya.

Hasil scraping akan disimpan format `json` sebagai berikut

```
{
    "website" : nama_website,
    "keyword" : keyword_pencarian,
    "scraping_start" : waktu_scraping_dimulai,
    "scraping_end" : waktu_scraping_berakhir,
    "article" : {
        "count" : jumlah_artikel,
        "data" : [
            {
                "title" : judul_artikel,
                "url" : url_artikel,
                "published_at" : waktu_publish_artikel,
                "full_content" : isi_artikel,
                "paragraph" : {
                    "count" : jumlah_paragraf_dalam_artikel,
                    "data" : {
                        "paragraf_1" : paragraf_ke_1,
                        ...
                        "paragtaf_n" : paragraf_ke_n
                    }
                }
            }, 
            ....
        ]
    }
}
```


```python
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
from datetime import datetime
import json
```

## Define Class

### Scraping Class

Merupakan SuperClass yang nantinya akan diextend oleh class lainnya. Class ini berisi method-method yang secara umum bisa diterapkan diseluruh website berita. Jika nantinya ada yang berbeda maka akan di override pada subclass


```python
class Scraping():
    def __init__(self, container_article, website, keyword=None):
        self.all_berita = []
        self.driver =  None
        self.container_article = container_article
        self.website = website
        self.keyword = keyword
    
    def get(self, url):
        """Membuat request dengan metode GET ke URL tertentu"""
        try: self.driver.get(url)
        except:pass
        
    def split_paragraph(self, p_elements):
        """Membaca seluruh paragraf disatu halaman republika"""
        for p in p_elements:
            text = p.text.strip()
            if text:
                self._paragraphs[f"paragraf_{self._paragraphs_count}"] = text
                self._paragraphs_count += 1
                
    def get_p_elements(self):
        """Mencari semua tag <p> yang merupakan bagian dari artikel"""
        return self.driver.find_elements(by=By.CSS_SELECTOR, value=f".{self.container_article} p")
    
    
    def _init_paragraph(self):
        """Inisialisasi jumlah paragraf dalam 1 artikel"""
        self._paragraphs = {}
        self._paragraphs_count = 1
    
        
    def _scraping(self, list_link, verbose=0, time_out=10, callback=None):
        """
        Lakukan scarping berdasarkan url yang ada di variabel
        list_link. Verbose selain 1 tidak akan melakukan print
        url yang sedang di scaping di console. Time out berguna
        untuk mengatur waktu loading maksmal dari website dalam
        satuan detik. Variabel list_link harus bertipe list python
        """
        self.driver = webdriver.Chrome()
        self.driver.set_page_load_timeout(time_out)
        
        for link in list_link:
            if verbose == 1:
                print(link)
            
            try:
                self.get(link)
                
                # method get_info_article harus didefinisikan oleh class
                # yang mengextend class Scarping
                published_date, title = self.get_info_article()
                self._init_paragraph()
                self.split_paragraph(self.get_p_elements())

                # callback jika satu berita ditampilkan dalam beberapa halaman
                if callback is not None: callback(link)

                self.all_berita.append({
                    'title' : title,
                    'url' : link,
                    'published_at' : published_date,
                    'full_content' : " ".join([self._paragraphs[p] for p in self._paragraphs.keys()]),
                    "paragraph" : {
                        "count" : self._paragraphs_count - 1,
                        "data" : self._paragraphs
                    }
                })
            except Exception as e:
                print(f"{link}. {str(e)}")
                
        self.driver.close()
        
    def save_data(self, folder):
        """Save data to file json"""
        data = {
            "website" : self.website,
            "keyword" : self.keyword,
            "scraping_start" : self.scraping_start,
            "scraping_end" : self.scraping_end,
            "article" : {
                "count" : len(self.all_berita),
                "data" : self.all_berita
            }        
        }

        with open(f"{folder}/{self.website_name}-{self.keyword}.json", 'w') as fp:
            json.dump(data, fp)
    
    def run(self, list_link, verbose=0, time_out=3, callback=None):
        """ Method ini yang di overiding di subclass jika memang diperlukan"""
        self.scraping_start = datetime.today().strftime("%Y-%m-%d %H:%M:%S")
        self._scraping(list_link, verbose, time_out, callback)
        self.scraping_end = datetime.today().strftime("%Y-%m-%d %H:%M:%S")
```

### Republika Class


```python
class ScrapingRepublika(Scraping):
    def __init__(self, keyword):
        super().__init__("artikel", "https://republika.co.id/", keyword)
        self.website_name = "republika"
    
    def get_info_article(self):
        """Ambil informasi waktu publish dan judul"""
        published_date = self.driver.find_element(by=By.CSS_SELECTOR, value='.wrap_detail_set .date_detail p').text
        title = self.driver.find_element(by=By.CSS_SELECTOR, value='.wrap_detail_set h1').text
        return published_date, title
    
    def has_pagination(self, link):
        """Ada beberapa artikel yang terpecah menjadi beberapa halaman"""
        try:
            self.driver.find_element(by=By.CLASS_NAME, value="pagination")
            part = 1
            while True:
                self.get(f"{link}-part{part}")
                p_elements = self.get_p_elements()

                if len(p_elements) == 0: break
                self.split_paragraph(p_elements)
                part += 1
        except:
            pass
        
    def run(self, list_link, verbose=0, time_out=3):
        """Overriding karena memakai callback has pagination"""
        self.scraping_start = datetime.today().strftime("%Y-%m-%d %H:%M:%S")
        self._scraping(list_link, verbose, time_out, callback=self.has_pagination)
        self.scraping_end = datetime.today().strftime("%Y-%m-%d %H:%M:%S")
```

### Kompas Class


```python
class ScrapingKompas(Scraping):
    def __init__(self, keyword):
        super().__init__("read__content", "https://www.kompas.com/", keyword)
        self.website_name = "kompas"
    
    
    def get_info_article(self):
        """Ambil informasi waktu publish dan judul"""
        published_date = self.driver.find_element(by=By.CSS_SELECTOR, value='.read__time').text.replace("Kompas.com - ", "")
        title = self.driver.find_element(by=By.CSS_SELECTOR, value='.read__title').text
        return published_date, title
    
        
    def run(self, list_link, verbose=0, time_out=3):
        """Override method karena link perlu ditambahkan query string
        page=all diakhir setiap link agar web menampilkan berita dalam
        satu halaman full"""
        
        new_list_link = []
        for link in list_link:
            new_list_link.append(f"{link}?page=all")
        
        self.scraping_start = datetime.today().strftime("%Y-%m-%d %H:%M:%S")
        self._scraping(new_list_link, verbose, time_out)
        self.scraping_end = datetime.today().strftime("%Y-%m-%d %H:%M:%S")
```

### Detik Class


```python
class ScrapingDetik(Scraping):
    def __init__(self, keyword):
        super().__init__("detail__body-text", "http://detik.com/", keyword)
        self.website_name = "detik"
                    
    def get_info_article(self):
        """Ambil informasi waktu publish dan judul"""
        published_date = self.driver.find_element(by=By.CSS_SELECTOR, value='.detail__date').text
        title = self.driver.find_element(by=By.CSS_SELECTOR, value='h1.detail__title').text
        return published_date, title
    
    def get_p_elements(self):
        """Mencari semua tag <p> dan div yang merupakan bagian dari artikel"""
        p = self.driver.find_elements(by=By.CSS_SELECTOR, value=f".{self.container_article} p")
        p += self.driver.find_elements(by=By.CSS_SELECTOR, value=f'.{self.container_article} div[style="text-align: left;"]')
        return p
    
    def scraping_title(self):
        self.driver = webdriver.Chrome()
        self.driver.get(self.website)
        self.driver.find_element(by=By.CSS_SELECTOR, value='input[placeholder="Cari Berita"]').send_keys(self.keyword, Keys.ENTER)

        link_detik = []
        while True:
            a_elements = self.driver.find_elements(by=By.CSS_SELECTOR, value=".list-berita article a")
            for a in a_elements:
                link_detik.append(a.get_attribute("href") + "?single=1")
                
            button = self.driver.find_elements(by=By.CSS_SELECTOR, value='.paging.text_center a')[-1]
            if button.get_attribute("href") != self.driver.current_url: button.click()
            else: break
                
        self.driver.close()
        return link_detik
        
        
    def run(self, verbose=0, time_out=3):
        """
        Overriding method karena perlu lakukan scraping title terlebih dahulu
        """
        self.scraping_start = datetime.today().strftime("%Y-%m-%d %H:%M:%S")
        self.list_link = self.scraping_title()
        self._scraping(self.list_link, verbose, time_out)
        self.scraping_end = datetime.today().strftime("%Y-%m-%d %H:%M:%S")
```

### Helper Function


```python
def read_link_berita(path):
    """Membaca file link berita yang memiliki format
    link1,link2,link3,....linkn
    """
    with open(path, 'r') as f:
        link_kompas = f.read().split(",")
        return list(set(link_kompas)) # remove duplicate
```

## Main

Dicontohkan melakukan scraping data dengan keyword `Permendikbud No 30 Tahun 2021`. Untuk `republika` dan `kompas` pengumpulan linknya dilakukan secara manual dan disimpan dalam file `data/list-berita-republika.txt` dan `data/list-berita-kompas.txt`.

Hasil scrapingnya akan disimpan dalam folder `data`


```python
link_republika = read_link_berita('data/list-berita-republika.txt')
republika = ScrapingRepublika("Permendikbud No 30 Tahun 2021")
republika.run(link_republika, verbose=0)
republika.save_data("data")
```


```python
link_kompas = read_link_berita('data/list-berita-kompas.txt')
kompas = ScrapingKompas("Permendikbud No 30 Tahun 2021")
kompas.run(link_kompas, verbose=0)
kompas.save_data('data')
```


```python
detik = ScrapingDetik("Permendikbud No 30 Tahun 2021")
detik.run(verbose=0, time_out=20)
detik.save_data('data')
```
