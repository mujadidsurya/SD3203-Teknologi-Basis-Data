# Mujadid Choirus Surya
# 121450015
# RA

# **Three Ways of Storing and Accessing Lots of Images in Python**

# Setup
## A Dataset to Play With
1. **Mengenal Dataset CIFAR-10**: Dataset ini terdiri dari 60.000 gambar berwarna dengan resolusi 32x32 piksel yang dikelompokkan ke dalam sepuluh kelas objek berbeda seperti anjing, kucing, dan pesawat. Ini adalah bagian dari dataset TinyImages yang lebih besar.

2. **Unduh Dataset CIFAR-10**: Anda dapat mengunduh dataset CIFAR-10 dalam versi Python melalui tautan yang disediakan di artikel. Ingat bahwa dataset ini akan membutuhkan sekitar 163MB ruang disk Anda.

3. **Ekstrak dan Pahami Format File**: Setelah diunduh dan diekstrak, Anda akan menemukan bahwa file-file dalam dataset tidak dalam format gambar yang dapat langsung dibaca oleh manusia. File-file tersebut telah diserialisasi dan disimpan dalam batch menggunakan cPickle.

4. **Memuat Dataset ke dalam Array NumPy**: Gunakan kode Python untuk mengurai ("unpickle") masing-masing dari lima file batch dan memuat semua gambar ke dalam array NumPy. Meskipun artikel tersebut tidak membahas secara mendalam tentang penggunaan modul pickle atau cPickle, penting untuk diketahui bahwa modul ini memungkinkan serialisasi objek Python apa pun tanpa kode tambahan.

## Setup for Storing Images on Disk
1. **Instal Python 3.x**: Pastikan bahwa Python 3.x telah terinstal pada sistem Anda.

2. **Instal Pillow**: Gunakan Pillow untuk manipulasi gambar. Anda dapat menginstalnya menggunakan pip: 
  ```
  pip install pillow
  ```
![alt text](image-1.png)

## Getting Started With LMDB
- LMDB adalah singkatan dari Lightning Memory-Mapped Database, sebuah penyimpanan kunci-nilai yang menggunakan file yang dipetakan ke memori.
- Menggunakan struktur pohon B+ untuk efisiensi pemetaan kunci-nilai dan penelusuran yang cepat.
- Keefisienan LMDB sangat bergantung pada pemetaan memori, yang memungkinkan akses data yang lebih cepat.
- Untuk menginstal binding Python untuk perpustakaan LMDB C, gunakan perintah:
  ```
  pip install lmdb
  ```
![alt text](image-2.png)

## Getting Started With HDF5
- HDF5 adalah singkatan dari Hierarchical Data Format versi 5, merupakan evolusi dari HDF4 dan merupakan versi yang saat ini dipelihara.
- File HDF5 dapat mengandung dua jenis objek: Dataset dan Grup.
  - **Dataset**: Array multidimensi dengan tipe dan ukuran seragam.
  - **Grup**: Wadah yang dapat menyimpan dataset atau grup lain, memungkinkan struktur data bersarang.
- Untuk bekerja dengan HDF5 di Python, instal pustaka yang diperlukan:
  ```
  pip install h5py
  ```
![alt text](image-3.png)

# Storing a Single Image
1. **Buat Direktori untuk Setiap Metode Penyimpanan**:  
   Buat tiga direktori terpisah untuk setiap metode penyimpanan yang akan diuji, yaitu sistem file standar, LMDB, dan format HDF5.  
   ```bash
   mkdir data/disk
   mkdir data/lmdb
   mkdir data/hdf5
   ```
2. **Simpan Jalur Direktori ke dalam Variabel Python**:
```python
from pathlib import Path
disk_dir = Path("data/disk/")
lmdb_dir = Path("data/lmdb/")
hdf5_dir = Path("data/hdf5/")
```
3. **Persiapan Data Gambar untuk Eksperimen**
Jika menggunakan dataset CIFAR-10 yang terdiri dari 50,000 gambar, gunakan setiap gambar dua kali untuk mendapatkan total 100,000 gambar dalam eksperimen.
4. **Menyusun Eksperimen dengan Berbagai Jumlah File**
Bandingkan kinerja metode penyimpanan dengan menguji berbagai jumlah gambar, mulai dari satu gambar hingga 100,000 gambar.

## Storing to Disk
1. **Simpan Gambar ke Disk**
   - **Inisialisasi** gambar yang sudah ada di memori sebagai array NumPy.
   - **Gunakan paket Pillow** yang telah diinstal sebelumnya untuk menyimpan gambar.
   - **Nama File**: Simpan gambar dengan format `.png` menggunakan ID gambar unik `image_id`.
   
```python
from PIL import Image
import csv

def store_single_disk(image, image_id, label):
    """ Stores a single image as a .png file on disk.
        Parameters:
        ---------------
        image       image array, (32, 32, 3) to be stored
        image_id    integer unique ID for image
        label       image label
    """
    Image.fromarray(image).save(disk_dir / f"{image_id}.png")

    with open(disk_dir / f"{image_id}.csv", "wt") as csvfile:
        writer = csv.writer(
            csvfile, delimiter=" ", quotechar="|", quoting=csv.QUOTE_MINIMAL
        )
        writer.writerow([label])
```

2. **Simpan Meta Data**
   - **Pertimbangkan Metode Penyimpanan**: Ada beberapa cara untuk menyimpan meta data.
   - **Enkode Label dalam Nama Gambar**: Metode ini menghindari penggunaan file tambahan tetapi membuat pengelolaan file lebih rumit.
   - **Simpan Label dalam File Terpisah**:
     - Buat file `.csv` untuk menyimpan label. Ini memungkinkan Anda mengelola label tanpa harus memuat gambar.

## Storing to LMDB

1. **Pengertian Dasar LMDB**
   - LMDB adalah sistem penyimpanan kunci-nilai di mana setiap entri disimpan sebagai array byte.
   - Kunci adalah pengenal unik untuk setiap gambar, dan nilai adalah gambar itu sendiri.
   - Baik kunci maupun nilai harus berupa string, jadi umumnya nilai diserialisasi menjadi string dan kemudian deserialisasi saat dibaca kembali.

2. **Serialisasi Menggunakan Pickle**
   - Gunakan modul `pickle` untuk serialisasi. Anda dapat menyertakan meta data gambar dalam basis data juga.
   - Ini menghindari kerumitan menghubungkan kembali meta data dengan data gambar saat memuat dataset dari disk.

3. **Membuat Kelas Python untuk Gambar dan Meta Data**
```python
class CIFAR_Image:
    def __init__(self, image, label):
        # Dimensions of image for reconstruction - not really necessary 
        # for this dataset, but some datasets may include images of 
        # varying sizes
        self.channels = image.shape[2]
        self.size = image.shape[:2]

        self.image = image.tobytes()
        self.label = label

    def get_image(self):
        """ Returns the image as a numpy array. """
        image = np.frombuffer(self.image, dtype=np.uint8)
        return image.reshape(*self.size, self.channels)
```
4. **Menetapkan Ukuran Peta**
    - Karena LMDB menggunakan memori-mapped, penting untuk menentukan berapa banyak memori yang diharapkan digunakan.
    - LMDB menggunakan variabel yang disebut map_size untuk ini.
5. **Operasi Transaksi**
    - Operasi baca dan tulis dengan LMDB dilakukan dalam transaksi, mirip dengan database tradisional.
    -  Transaksi terdiri dari sekelompok operasi pada database.
```python
import lmdb
import pickle

def store_single_lmdb(image, image_id, label):
    """ Stores a single image to a LMDB.
        Parameters:
        ---------------
        image       image array, (32, 32, 3) to be stored
        image_id    integer unique ID for image
        label       image label
    """
    map_size = image.nbytes * 10

    # Create a new LMDB environment
    env = lmdb.open(str(lmdb_dir / f"single_lmdb"), map_size=map_size)

    # Start a new write transaction
    with env.begin(write=True) as txn:
        # All key-value pairs need to be strings
        value = CIFAR_Image(image, label)
        key = f"{image_id:08}"
        txn.put(key.encode("ascii"), pickle.dumps(value))
    env.close()
```


## Storing With HDF5

1. **Pemahaman File HDF5**
   - Ingatlah bahwa file HDF5 dapat berisi lebih dari satu dataset. Ini memungkinkan penyimpanan yang efisien dari berbagai jenis data.

2. **Membuat Dua Dataset**
   - Anda akan membuat dua dataset dalam file HDF5: satu untuk gambar dan satu lagi untuk meta data gambar.

3. **Menentukan Tipe Data**
   - Gunakan `h5py.h5t.STD_U8BE` untuk menentukan tipe data yang akan disimpan dalam dataset. Tipe data ini adalah integer 8-bit tak bertanda.
   - Tipe data ini dipilih karena sesuai untuk menyimpan gambar yang diwakili dalam bentuk pixel values yang berkisar dari 0 sampai 255.

4. **Pertimbangan dalam Memilih Tipe Data**
   - Pilihan tipe data sangat mempengaruhi kebutuhan runtime dan penyimpanan dari file HDF5.
   - Pilihlah tipe data yang memenuhi kebutuhan minimal Anda untuk mengoptimalkan penggunaan sumber daya.

```python
import h5py

def store_single_hdf5(image, image_id, label):
    """ Stores a single image to an HDF5 file.
        Parameters:
        ---------------
        image       image array, (32, 32, 3) to be stored
        image_id    integer unique ID for image
        label       image label
    """
    # Create a new HDF5 file
    file = h5py.File(hdf5_dir / f"{image_id}.h5", "w")

    # Create a dataset in the file
    dataset = file.create_dataset(
        "image", np.shape(image), h5py.h5t.STD_U8BE, data=image
    )
    meta_set = file.create_dataset(
        "meta", np.shape(label), h5py.h5t.STD_U8BE, data=label
    )
    file.close()
```

## Experiments for Storing a Single Image
1. **Menyiapkan Fungsi Penyimpanan**
   - Simpan semua fungsi penyimpanan dalam satu kamus untuk memudahkan pemanggilan selama eksperimen waktu.
```python
_store_single_funcs = dict(
    disk=store_single_disk, lmdb=store_single_lmdb, hdf5=store_single_hdf5
)
```

2. **Melakukan Eksperimen Waktu**
    - Jalankan eksperimen untuk menyimpan gambar pertama dari CIFAR dan labelnya dalam tiga cara yang berbeda: Disk, LMDB, dan HDF5.
    - Catat waktu runtime dan penggunaan memori untuk setiap metode.

```python
from timeit import timeit
store_single_timings = dict()

for method in ("disk", "lmdb", "hdf5"):
    t = timeit(
        "_store_single_funcs[method](image, 0, label)",
        setup="image=images[0]; label=labels[0]",
        number=1,
        globals=globals(),
    )
    store_single_timings[method] = t
    print(f"Method: {method}, Time usage: {t}")
```
3. **Hasil Eksperimen**
    - Berikut adalah hasil dari eksperimen yang menunjukkan waktu penyimpanan
![alt text](image-4.png)

# Storing Many Images
## Adjusting the Code for Many Images
1. **Pengenalan Metode Penyimpanan**
   - Menyimpan banyak gambar sebagai file .png adalah sederhana seperti memanggil `store_single_method()` beberapa kali untuk metode Disk.
   - Namun, untuk LMDB atau HDF5, Anda tidak ingin membuat file database terpisah untuk setiap gambar. Sebaliknya, Anda ingin menyimpan semua gambar dalam satu atau beberapa file.

2. **Modifikasi Fungsi untuk Menyimpan Banyak Gambar**
   - Anda perlu mengubah kode dan membuat tiga fungsi baru yang menerima beberapa gambar: `store_many_disk()`, `store_many_lmdb()`, dan `store_many_hdf5()`.

3. **Menyimpan Gambar ke Disk**
   - **Fungsi `store_many_disk()`:** Metode ini diubah untuk melakukan loop atas setiap gambar dalam daftar dan menyimpannya sebagai file terpisah.

4. **Menyimpan Gambar Menggunakan LMDB**
   - **Fungsi `store_many_lmdb()`:** Loop juga diperlukan di sini karena kita membuat objek `CIFAR_Image` untuk setiap gambar dan meta datanya.


5. **Menyimpan Gambar Menggunakan HDF5**
   - **Fungsi `store_many_hdf5()`:** Penyesuaian terkecil ada pada metode ini. Faktanya, hampir tidak ada penyesuaian sama sekali! File HDF5 tidak memiliki batasan ukuran file selain pembatasan eksternal atau ukuran dataset, sehingga semua gambar dimasukkan ke dalam satu dataset, sama seperti sebelumnya.

```python
def store_many_disk(images, labels):
    """ Stores an array of images to disk
        Parameters:
        ---------------
        images       images array, (N, 32, 32, 3) to be stored
        labels       labels array, (N, 1) to be stored
    """
    num_images = len(images)

    # Save all the images one by one
    for i, image in enumerate(images):
        Image.fromarray(image).save(disk_dir / f"{i}.png")

    # Save all the labels to the csv file
    with open(disk_dir / f"{num_images}.csv", "w") as csvfile:
        writer = csv.writer(
            csvfile, delimiter=" ", quotechar="|", quoting=csv.QUOTE_MINIMAL
        )
        for label in labels:
            # This typically would be more than just one value per row
            writer.writerow([label])

def store_many_lmdb(images, labels):
    """ Stores an array of images to LMDB.
        Parameters:
        ---------------
        images       images array, (N, 32, 32, 3) to be stored
        labels       labels array, (N, 1) to be stored
    """
    num_images = len(images)

    map_size = num_images * images[0].nbytes * 10

    # Create a new LMDB DB for all the images
    env = lmdb.open(str(lmdb_dir / f"{num_images}_lmdb"), map_size=map_size)

    # Same as before — but let's write all the images in a single transaction
    with env.begin(write=True) as txn:
        for i in range(num_images):
            # All key-value pairs need to be Strings
            value = CIFAR_Image(images[i], labels[i])
            key = f"{i:08}"
            txn.put(key.encode("ascii"), pickle.dumps(value))
    env.close()

def store_many_hdf5(images, labels):
    """ Stores an array of images to HDF5.
        Parameters:
        ---------------
        images       images array, (N, 32, 32, 3) to be stored
        labels       labels array, (N, 1) to be stored
    """
    num_images = len(images)

    # Create a new HDF5 file
    file = h5py.File(hdf5_dir / f"{num_images}_many.h5", "w")

    # Create a dataset in the file
    dataset = file.create_dataset(
        "images", np.shape(images), h5py.h5t.STD_U8BE, data=images
    )
    meta_set = file.create_dataset(
        "meta", np.shape(labels), h5py.h5t.STD_U8BE, data=labels
    )
    file.close()
```
## Preparing the Dataset
Sebelum menjalankan eksperimen lagi, mari kita pertama kali menggandakan ukuran dataset kita agar kita dapat menguji dengan hingga 100.000 gambar. 
```python
cutoffs = [10, 100, 1000, 10000, 100000]

# Let's double our images so that we have 100,000
images = np.concatenate((images, images), axis=0)
labels = np.concatenate((labels, labels), axis=0)

# Make sure you actually have 100,000 images and labels
print(np.shape(images))
print(np.shape(labels))
```
Setelah menggandakan ukuran dataset, Anda dapat menjalankan eksperimen dengan dataset yang baru saja diperbesar untuk menguji performa metode penyimpanan dengan jumlah gambar hingga 100.000.

## Experiment for Storing Many Images
```python
_store_many_funcs = dict(
    disk=store_many_disk, lmdb=store_many_lmdb, hdf5=store_many_hdf5
)

from timeit import timeit

store_many_timings = {"disk": [], "lmdb": [], "hdf5": []}

for cutoff in cutoffs:
    for method in ("disk", "lmdb", "hdf5"):
        t = timeit(
            "_store_many_funcs[method](images_, labels_)",
            setup="images_=images[:cutoff]; labels_=labels[:cutoff]",
            number=1,
            globals=globals(),
        )
        store_many_timings[method].append(t)

        # Print out the method, cutoff, and elapsed time
        print(f"Method: {method}, Time usage: {t}")
```
Perlu bersabar sejenak dan menunggu dengan penuh penasaran sementara 111,110 gambar disimpan tiga kali masing-masing ke disk Anda, dalam tiga format yang berbeda. Anda juga perlu bersiap untuk mengucapkan selamat tinggal pada sekitar 2 GB ruang disk.

Grafik pertama menunjukkan waktu penyimpanan normal, tanpa penyesuaian, menyoroti perbedaan drastis antara menyimpan ke file .png dan ke LMDB atau HDF5.
![alt text](image-5.png)
Grafik kedua menunjukkan log dari waktu yang diukur, menyoroti bahwa HDF5 mulai lebih lambat dari LMDB tetapi, dengan jumlah gambar yang lebih besar, akhirnya sedikit lebih unggul.
![alt text](image-6.png)


# Reading a Single Image
## Reading From Disk
- Buka file gambar (.png) menggunakan identifikasi gambar (`image_id`).
- Buka dan baca file CSV untuk menemukan metadata yang sesuai dengan `image_id`.

```python
def read_single_disk(image_id):
    """ Stores a single image to disk.
        Parameters:
        ---------------
        image_id    integer unique ID for image

        Returns:
        ----------
        image       image array, (32, 32, 3) to be stored
        label       associated meta data, int label
    """
    image = np.array(Image.open(disk_dir / f"{image_id}.png"))

    with open(disk_dir / f"{image_id}.csv", "r") as csvfile:
        reader = csv.reader(
            csvfile, delimiter=" ", quotechar="|", quoting=csv.QUOTE_MINIMAL
        )
        label = int(next(reader)[0])

    return image, label
```
## Reading From LMDB
- Buka database LMDB menggunakan path yang diberikan.
- Gunakan `image_id` untuk mendapatkan data gambar yang diserialkan.
- Deserialkan data untuk mendapatkan gambar dan metadata.

```python
def read_single_lmdb(image_id):
    """ Stores a single image to LMDB.
        Parameters:
        ---------------
        image_id    integer unique ID for image

        Returns:
        ----------
        image       image array, (32, 32, 3) to be stored
        label       associated meta data, int label
    """
    # Open the LMDB environment
    env = lmdb.open(str(lmdb_dir / f"single_lmdb"), readonly=True)

    # Start a new read transaction
    with env.begin() as txn:
        # Encode the key the same way as we stored it
        data = txn.get(f"{image_id:08}".encode("ascii"))
        # Remember it's a CIFAR_Image object that is loaded
        cifar_image = pickle.loads(data)
        # Retrieve the relevant bits
        image = cifar_image.get_image()
        label = cifar_image.label
    env.close()

    return image, label
```

## Reading From HDF5
- Buka file HDF5.
- Akses dataset `images` untuk mendapatkan gambar, dan dataset `labels` untuk metadata, menggunakan indeks gambar (`image_index`).

```python
def read_single_hdf5(image_id):
    """ Stores a single image to HDF5.
        Parameters:
        ---------------
        image_id    integer unique ID for image

        Returns:
        ----------
        image       image array, (32, 32, 3) to be stored
        label       associated meta data, int label
    """
    # Open the HDF5 file
    file = h5py.File(hdf5_dir / f"{image_id}.h5", "r+")

    image = np.array(file["/image"]).astype("uint8")
    label = int(np.array(file["/meta"]).astype("uint8"))

    return image, label

_read_single_funcs = dict(
    disk=read_single_disk, lmdb=read_single_lmdb, hdf5=read_single_hdf5
)
```

## Experiment for Reading a Single Image

1. Persiapan Eksperimen
    - Kode eksperimen digunakan untuk membaca satu gambar beserta metadata-nya menggunakan tiga metode berbeda.

2. Metode yang Digunakan
    - **Disk**: Membaca langsung dari disk.
    - **LMDB**: Menggunakan database LMDB.
    - **HDF5**: Menggunakan format data HDF5.

```python
from timeit import timeit

read_single_timings = dict()

for method in ("disk", "lmdb", "hdf5"):
    t = timeit(
        "_read_single_funcs[method](0)",
        setup="image=images[0]; label=labels[0]",
        number=1,
        globals=globals(),
    )
    read_single_timings[method] = t
    print(f"Method: {method}, Time usage: {t}")
```

3. Hasil Eksperimen
    - Berikut adalah hasil waktu yang dibutuhkan untuk membaca satu gambar beserta metadata-nya untuk setiap metode:
![alt text](image-7.png)

4. Kesimpulan
    - Membaca file .png dan .csv langsung dari disk sedikit lebih cepat dibandingkan metode lainnya.
    - Namun, ketiga metode ini memberikan performa yang cepat dan hampir sama, sehingga perbedaannya dianggap tidak signifikan.

# Reading Many Images
## Adjusting the Code for Many Images
1. Perluasan Fungsi Pembacaan
    - Perluas fungsi yang ada dengan menambahkan varian `read_many_`, yang memungkinkan pembacaan banyak gambar sekaligus.
    - Fungsi `read_many_` ini akan digunakan untuk eksperimen selanjutnya, menilai performa saat membaca jumlah gambar yang berbeda.

 2. Menyimpan Fungsi dalam Dictionary
    - Simpan fungsi pembacaan yang telah diperluas ke dalam dictionary, mirip dengan penyimpanan fungsi penulisan sebelumnya.
```python
def read_many_disk(num_images):
    """ Reads image from disk.
        Parameters:
        ---------------
        num_images   number of images to read

        Returns:
        ----------
        images      images array, (N, 32, 32, 3) to be stored
        labels      associated meta data, int label (N, 1)
    """
    images, labels = [], []

    # Loop over all IDs and read each image in one by one
    for image_id in range(num_images):
        images.append(np.array(Image.open(disk_dir / f"{image_id}.png")))

    with open(disk_dir / f"{num_images}.csv", "r") as csvfile:
        reader = csv.reader(
            csvfile, delimiter=" ", quotechar="|", quoting=csv.QUOTE_MINIMAL
        )
        for row in reader:
            labels.append(int(row[0]))
    return images, labels

def read_many_lmdb(num_images):
    """ Reads image from LMDB.
        Parameters:
        ---------------
        num_images   number of images to read

        Returns:
        ----------
        images      images array, (N, 32, 32, 3) to be stored
        labels      associated meta data, int label (N, 1)
    """
    images, labels = [], []
    env = lmdb.open(str(lmdb_dir / f"{num_images}_lmdb"), readonly=True)

    # Start a new read transaction
    with env.begin() as txn:
        # Read all images in one single transaction, with one lock
        # We could split this up into multiple transactions if needed
        for image_id in range(num_images):
            data = txn.get(f"{image_id:08}".encode("ascii"))
            # Remember that it's a CIFAR_Image object 
            # that is stored as the value
            cifar_image = pickle.loads(data)
            # Retrieve the relevant bits
            images.append(cifar_image.get_image())
            labels.append(cifar_image.label)
    env.close()
    return images, labels

def read_many_hdf5(num_images):
    """ Reads image from HDF5.
        Parameters:
        ---------------
        num_images   number of images to read

        Returns:
        ----------
        images      images array, (N, 32, 32, 3) to be stored
        labels      associated meta data, int label (N, 1)
    """
    images, labels = [], []

    # Open the HDF5 file
    file = h5py.File(hdf5_dir / f"{num_images}_many.h5", "r+")

    images = np.array(file["/images"]).astype("uint8")
    labels = np.array(file["/meta"]).astype("uint8")

    return images, labels

_read_many_funcs = dict(
    disk=read_many_disk, lmdb=read_many_lmdb, hdf5=read_many_hdf5
)
```

# Experiment for Reading Many Images
```python
from timeit import timeit

read_many_timings = {"disk": [], "lmdb": [], "hdf5": []}

for cutoff in cutoffs:
    for method in ("disk", "lmdb", "hdf5"):
        t = timeit(
            "_read_many_funcs[method](num_images)",
            setup="num_images=cutoff",
            number=0,
            globals=globals(),
        )
        read_many_timings[method].append(t)

        # Print out the method, cutoff, and elapsed time
        print(f"Method: {method}, No. images: {cutoff}, Time usage: {t}")

```
![alt text](image-11.png)

```python
disk_x_r = read_many_timings["disk"]
lmdb_x_r = read_many_timings["lmdb"]
hdf5_x_r = read_many_timings["hdf5"]

plot_with_legend(
    cutoffs,
    [disk_x_r, lmdb_x_r, hdf5_x_r],
    ["PNG files", "LMDB", "HDF5"],
    "Number of images",
    "Seconds to read",
    "Read time",
    log=False,
)

plot_with_legend(
    cutoffs,
    [disk_x_r, lmdb_x_r, hdf5_x_r],
    ["PNG files", "LMDB", "HDF5"],
    "Number of images",
    "Seconds to read",
    "Log read time",
    log=True,
)
```
![alt text](image-10.png)
![alt text](image-9.png)
### Grafik Atas: Waktu Baca Normal
- **Deskripsi**: Grafik ini menunjukkan waktu baca normal (tanpa penyesuaian) dari berbagai metode.
- **Temuan**: Terdapat perbedaan yang signifikan antara waktu membaca dari file .png dibandingkan dengan menggunakan LMDB atau HDF5.

### Grafik Bawah: Logaritma Waktu Baca
- **Deskripsi**: Grafik ini menampilkan logaritma dari waktu baca, yang memperjelas perbedaan relatif dengan jumlah gambar yang lebih sedikit.
- **Temuan**: Dapat dilihat bahwa HDF5 awalnya lebih lambat dibandingkan dengan LMDB, namun, dengan penambahan jumlah gambar, HDF5 menjadi secara konsisten lebih cepat dari LMDB dengan margin yang kecil.

### Kesimpulan
- **Perbandingan Metode**: Dari analisis kedua grafik, HDF5 menunjukkan performa yang membaik secara signifikan dengan peningkatan jumlah gambar, meskipun mulai dari posisi yang lebih lambat. Ini menunjukkan bahwa HDF5 mungkin lebih efisien dalam skenario pembacaan batch gambar yang lebih besar.
- **Pengaruh Jumlah Gambar**: Grafik logaritma waktu baca menggarisbawahi bahwa dengan sedikit gambar, perbedaan antara metode tidak sejelas saat menggunakan lebih banyak gambar.

```python
plot_with_legend(
    cutoffs,
    [disk_x_r, lmdb_x_r, hdf5_x_r, disk_x, lmdb_x, hdf5_x],
    [
        "Read PNG",
        "Read LMDB",
        "Read HDF5",
        "Write PNG",
        "Write LMDB",
        "Write HDF5",
    ],
    "Number of images",
    "Seconds",
    "Log Store and Read Times",
    log=False,
)
```
![alt text](image-12.png)