# Applied Cryptography: First Assignment
> @author: 徐元桢 Rhyne Hsu</br>
@email: yuanzhen.xu@stu.pku.edu.cn

## ZUC Stream Cypher
In this part we generate DDTs & LATs of S0 & S1 boxes in [ZUC Stream Cypher](https://github.com/guanzhi/GM-Standards/blob/master/GMT密码行标/GMT%200001.1-2012%20祖冲之序列密码算法第1部分：算法描述.pdf).

### Quick Start
To run this project, simply run `cargo run` in your terminal. Rust environment required, of course.

### Implementation
I'm using the **BEST** language in the world, namely [**RUST🦀**](https://www.rust-lang.org), to implement this lab. Codes are in `./src/main.rs` and are all self-explanatory.

To do this job, I defined a `Sbox` struct with two fields (`table`, `data`) and implemented `new`, `generate_ddt`, `generate_lat`, `save_csv` for it. (In rust🦀, we don't have `class`. Instead, we implement functions for a struct). 

Function prototypes of them are listed below:

| function prototype |
| :--- |
| `pub fn new(name: &str, data: [usize; 256]) -> Self` |
| `pub fn generate_ddt(&self, save_dir: &str)` |
| `pub fn generate_lat(&self, save_dir: &str)` |
| `fn save_csv(ddt: &Vec<Vec<i16>>, save_path: &str) -> Result<(), Box<dyn Error>>` |


Below is the main logic.
```rust
fn main() {
    // initialize Sbox objects: s0 & s1
    let s0 = Sbox::new("s0", [ 0x3e,0x72,0x5b, ... ]);
    let s1 = Sbox::new("s1", [ 0x55,0xc2,0x63, ... ]);

    // create directory to save results
    let save_dir = "./results";
    match fs::create_dir(save_dir) {
        Ok(_) => println!("Directory {save_dir} created"),
        Err(_) => eprint!("Directory {save_dir} already exists\n"),
    };

    s0.generate_ddt(save_dir);
    s1.generate_ddt(save_dir);

    s0.generate_lat(save_dir);
    s1.generate_lat(save_dir);
}
```

Details of DDT and LAT calculation is discussed in next parts.

### DDT Calculation
`i` and `j` represents first and second input to the S-box, iterating from 0 to 255 respectively.

Difference is computed as XOR result. So we compute the difference of (1: the two inputs) and (2: two outputs of them).

The `input_diff` is used as row index, `output_diff` is used as col index, and we increment corresponding DDT element.

The complete DDT is generated after all the loops.
```rust
for i in 0..256 {
    for j in 0..256 {
        let input_diff = i ^ j;
        let output_diff = self.table[i] ^ self.table[j];
        ddt[input_diff][output_diff] += 1;
    }
}
```

### LAT Calculation
We iterate `input_mask` and `output_mask` from 0 to 255, representing different combinations of input and output bits.

In each iteration, we count how many times the input and output parities match for all possible input value `x`. Then `input_mask` row and `output_mask` col element of LAT is updated as `cnt - 128`.

The complete LAT is generated after all the loops.
```rust
for input_mask in 0..256 {
    for output_mask in 0..256 {
        let mut cnt: i16 = 0;
        for x in 0..256 {
            let input_parity = (x as u8 & input_mask as u8).count_ones() % 2;
            let output_parity = (self.table[x] as u8 & output_mask as u8).count_ones() % 2;
            if input_parity == output_parity { cnt += 1; }
        }
        lat[input_mask][output_mask] = cnt - 128;
    }
}
```

### Result
Results are saved under `./results` directory.

| [DDT_s0.csv](./results/DDT_s0.csv) | [DDT_s1.csv](./results/DDT_s1.csv) | [LAT_s0.csv](./results/LAT_s0.csv) | [LAT_s1.csv](./results/LAT_s1.csv) |

## BasicSPN
> $\mathcal{Q}$: Why does the BasicSPN algorithm construct the last round in such a way that it does not include the permutation operation, and why does it XOR with round key at the end? 

$\mathcal{A}$: 
1. The absence of the permutation following the final round allows the decryption network to maintain the same structure as the encryption network. If a permutation followed the last substitution layer in the encryption process, the decryption would must have a permutation before the initial substitution layer.

2. The S-box is reversible, if we don't add a key mixing, i.e. XOR with the subkey, the S-box in the last round becomes trivial. By applying a subkey after the final round, we ensure the last layer of substitution would not be easily exploited by simply working backward through the last round’s substitution.

### Reference
- [*A Tutorial on Linear and Differential Cryptanalysis*](https://www.engr.mun.ca/~howard/PAPERS/ldc_tutorial.pdf) by Howard M. Heys