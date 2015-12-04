# meta_cpu
C++ meta-programming exercise implementing 3 address code at compile time

This program implents the following C code into a tempalte that is executable at compile time. Starting with 

        void print_primes(){
                struct local{
                        static bool test(int i){
                                for(int j=3;j*j<=i;j+=2){
                                        if( i % j == 0 )
                                                return false;
                                }
                                return true;
                        }
                };
                for(int i=3;i<=CONFIG_MAX_PRIME;i+=2){
                        if( local::test(i)){
                                std::cout << i << ",";
                        }
                }
                std::cout << std::endl;
        }

The above code finds all the primes except 2. Translating this into an unstructed assembaly like code 


        void print_primes_5(){
                std::array<int,4> reg;

                reg[0] = 3;
        start_outer:
                reg[2] = reg[0] - CONFIG_MAX_PRIME;
                if( reg[2] > 0 )
                        goto end_outer;
                reg[1]=3;
        start_inner:
                reg[2] = reg[1] * reg[1];
                reg[2] = reg[2] - reg[0];
                if( reg[2] > 0 )
                        goto end_inner;
                reg[2] = reg[0] % reg[1];
                if( reg[2] == 0 )
                        goto not_prime;
                reg[1] += 2;
                goto start_inner;
        end_inner:;
                std::cout << reg[0];
                std::cout << ",";
        not_prime:;
                reg[0]+=2;
                goto start_outer;
        end_outer:;
                std::cout << std::endl;
        }

This can then be expressed at the folloing template

        using prog_ = mpl::vector<
                mov< int_<3>, reg<0> >, 
                start_outer_,
                sub< reg<0>, int_<CONFIG_MAX_PRIME>, reg<2> >,
                jgt< reg<2>, end_outer_> , 
                mov< int_<3>, reg<1> >,
                start_inner_,
                mul< reg<1>, reg<1>, reg<2> >,
                sub< reg<2>, reg<0>, reg<2> >,
                jgt< reg<2>, end_inner_ >,
                mod< reg<0>, reg<1>, reg<2> >,
                jeq< reg<2>, not_prime_ >,
                add< reg<1>, int_<2>, reg<1> >,
                jmp<start_inner_>,
                end_inner_,
                push< reg<0> >,
                not_prime_,
                add< reg<0>, int_<2>, reg<0> >,
                jmp<start_outer_>,
                end_outer_
        >;

        ./mcpu
        ------- asm code
        0   mov 3, %0
        1   start_outer_:
        2   sub %0, 45, %2
        3   jgt %2, end_outer_
        4   mov 3, %1
        5   start_inner_:
        6   mul %1, %1, %2
        7   sub %2, %0, %2
        8   jgt %2, end_inner_
        9   mod %0, %1, %2
        10  jeq %2, not_prime_
        11  add %1, 2, %1
        12  jmp start_inner_
        13  end_inner_:
        14  push %0
        15  not_prime_:
        16  add %0, 2, %0
        17  jmp start_outer_
        18  end_outer_:
        ------- symbols resolved
        0   mov 3, %0
        1   sub %0, 45, %2
        2   jgt %2, 14
        3   mov 3, %1
        4   mul %1, %1, %2
        5   sub %2, %0, %2
        6   jgt %2, 11
        7   mod %0, %1, %2
        8   jeq %2, 12
        9   add %1, 2, %1
        10  jmp 4
        11  push %0
        12  add %0, 2, %0
        13  jmp 1
        14  end
        ------- debug run
        0   mov 3, %0       [  3,  0,  0],   1,   1  {}
        1   sub %0, 45, %2  [  3,  0,-42],   2,   1  {}
        2   jgt %2, 14      [  3,  0,-42],   3,   1  {}
        3   mov 3, %1       [  3,  3,-42],   4,   1  {}
        4   mul %1, %1, %2  [  3,  3,  9],   5,   1  {}
        5   sub %2, %0, %2  [  3,  3,  6],   6,   1  {}
        6   jgt %2, 11      [  3,  3,  6],  11,   1  {}
        7   push %0         [  3,  3,  6],  12,   1  {3}
        8   add %0, 2, %0   [  5,  3,  6],  13,   1  {3}
        9   jmp 1           [  5,  3,  6],   1,   1  {3}
        10  sub %0, 45, %2  [  5,  3,-40],   2,   1  {3}
        11  jgt %2, 14      [  5,  3,-40],   3,   1  {3}
        12  mov 3, %1       [  5,  3,-40],   4,   1  {3}
        13  mul %1, %1, %2  [  5,  3,  9],   5,   1  {3}
        14  sub %2, %0, %2  [  5,  3,  4],   6,   1  {3}
        15  jgt %2, 11      [  5,  3,  4],  11,   1  {3}
        16  push %0         [  5,  3,  4],  12,   1  {5,3}
        17  add %0, 2, %0   [  7,  3,  4],  13,   1  {5,3}
        18  jmp 1           [  7,  3,  4],   1,   1  {5,3}
        19  sub %0, 45, %2  [  7,  3,-38],   2,   1  {5,3}
        20  jgt %2, 14      [  7,  3,-38],   3,   1  {5,3}
        21  mov 3, %1       [  7,  3,-38],   4,   1  {5,3}
        22  mul %1, %1, %2  [  7,  3,  9],   5,   1  {5,3}
        23  sub %2, %0, %2  [  7,  3,  2],   6,   1  {5,3}
        24  jgt %2, 11      [  7,  3,  2],  11,   1  {5,3}
        25  push %0         [  7,  3,  2],  12,   1  {7,5,3}
        26  add %0, 2, %0   [  9,  3,  2],  13,   1  {7,5,3}
        27  jmp 1           [  9,  3,  2],   1,   1  {7,5,3}
        28  sub %0, 45, %2  [  9,  3,-36],   2,   1  {7,5,3}
        29  jgt %2, 14      [  9,  3,-36],   3,   1  {7,5,3}
        30  mov 3, %1       [  9,  3,-36],   4,   1  {7,5,3}
        31  mul %1, %1, %2  [  9,  3,  9],   5,   1  {7,5,3}
        32  sub %2, %0, %2  [  9,  3,  0],   6,   1  {7,5,3}
        33  jgt %2, 11      [  9,  3,  0],   7,   1  {7,5,3}
        34  mod %0, %1, %2  [  9,  3,  0],   8,   1  {7,5,3}
        35  jeq %2, 12      [  9,  3,  0],  12,   1  {7,5,3}
        36  add %0, 2, %0   [ 11,  3,  0],  13,   1  {7,5,3}
        37  jmp 1           [ 11,  3,  0],   1,   1  {7,5,3}
        38  sub %0, 45, %2  [ 11,  3,-34],   2,   1  {7,5,3}
        39  jgt %2, 14      [ 11,  3,-34],   3,   1  {7,5,3}
        40  mov 3, %1       [ 11,  3,-34],   4,   1  {7,5,3}
        41  mul %1, %1, %2  [ 11,  3,  9],   5,   1  {7,5,3}
        42  sub %2, %0, %2  [ 11,  3, -2],   6,   1  {7,5,3}
        43  jgt %2, 11      [ 11,  3, -2],   7,   1  {7,5,3}
        44  mod %0, %1, %2  [ 11,  3,  2],   8,   1  {7,5,3}
        45  jeq %2, 12      [ 11,  3,  2],   9,   1  {7,5,3}
        46  add %1, 2, %1   [ 11,  5,  2],  10,   1  {7,5,3}
        47  jmp 4           [ 11,  5,  2],   4,   1  {7,5,3}
        48  mul %1, %1, %2  [ 11,  5, 25],   5,   1  {7,5,3}
        49  sub %2, %0, %2  [ 11,  5, 14],   6,   1  {7,5,3}
        50  jgt %2, 11      [ 11,  5, 14],  11,   1  {7,5,3}
        51  push %0         [ 11,  5, 14],  12,   1  {11,7,5,3}
        52  add %0, 2, %0   [ 13,  5, 14],  13,   1  {11,7,5,3}
        53  jmp 1           [ 13,  5, 14],   1,   1  {11,7,5,3}
        54  sub %0, 45, %2  [ 13,  5,-32],   2,   1  {11,7,5,3}
        55  jgt %2, 14      [ 13,  5,-32],   3,   1  {11,7,5,3}
        56  mov 3, %1       [ 13,  3,-32],   4,   1  {11,7,5,3}
        57  mul %1, %1, %2  [ 13,  3,  9],   5,   1  {11,7,5,3}
        58  sub %2, %0, %2  [ 13,  3, -4],   6,   1  {11,7,5,3}
        59  jgt %2, 11      [ 13,  3, -4],   7,   1  {11,7,5,3}
        60  mod %0, %1, %2  [ 13,  3,  1],   8,   1  {11,7,5,3}
        61  jeq %2, 12      [ 13,  3,  1],   9,   1  {11,7,5,3}
        62  add %1, 2, %1   [ 13,  5,  1],  10,   1  {11,7,5,3}
        63  jmp 4           [ 13,  5,  1],   4,   1  {11,7,5,3}
        64  mul %1, %1, %2  [ 13,  5, 25],   5,   1  {11,7,5,3}
        65  sub %2, %0, %2  [ 13,  5, 12],   6,   1  {11,7,5,3}
        66  jgt %2, 11      [ 13,  5, 12],  11,   1  {11,7,5,3}
        67  push %0         [ 13,  5, 12],  12,   1  {13,11,7,5,3}
        68  add %0, 2, %0   [ 15,  5, 12],  13,   1  {13,11,7,5,3}
        69  jmp 1           [ 15,  5, 12],   1,   1  {13,11,7,5,3}
        70  sub %0, 45, %2  [ 15,  5,-30],   2,   1  {13,11,7,5,3}
        71  jgt %2, 14      [ 15,  5,-30],   3,   1  {13,11,7,5,3}
        72  mov 3, %1       [ 15,  3,-30],   4,   1  {13,11,7,5,3}
        73  mul %1, %1, %2  [ 15,  3,  9],   5,   1  {13,11,7,5,3}
        74  sub %2, %0, %2  [ 15,  3, -6],   6,   1  {13,11,7,5,3}
        75  jgt %2, 11      [ 15,  3, -6],   7,   1  {13,11,7,5,3}
        76  mod %0, %1, %2  [ 15,  3,  0],   8,   1  {13,11,7,5,3}
        77  jeq %2, 12      [ 15,  3,  0],  12,   1  {13,11,7,5,3}
        78  add %0, 2, %0   [ 17,  3,  0],  13,   1  {13,11,7,5,3}
        79  jmp 1           [ 17,  3,  0],   1,   1  {13,11,7,5,3}
        80  sub %0, 45, %2  [ 17,  3,-28],   2,   1  {13,11,7,5,3}
        81  jgt %2, 14      [ 17,  3,-28],   3,   1  {13,11,7,5,3}
        82  mov 3, %1       [ 17,  3,-28],   4,   1  {13,11,7,5,3}
        83  mul %1, %1, %2  [ 17,  3,  9],   5,   1  {13,11,7,5,3}
        84  sub %2, %0, %2  [ 17,  3, -8],   6,   1  {13,11,7,5,3}
        85  jgt %2, 11      [ 17,  3, -8],   7,   1  {13,11,7,5,3}
        86  mod %0, %1, %2  [ 17,  3,  2],   8,   1  {13,11,7,5,3}
        87  jeq %2, 12      [ 17,  3,  2],   9,   1  {13,11,7,5,3}
        88  add %1, 2, %1   [ 17,  5,  2],  10,   1  {13,11,7,5,3}
        89  jmp 4           [ 17,  5,  2],   4,   1  {13,11,7,5,3}
        90  mul %1, %1, %2  [ 17,  5, 25],   5,   1  {13,11,7,5,3}
        91  sub %2, %0, %2  [ 17,  5,  8],   6,   1  {13,11,7,5,3}
        92  jgt %2, 11      [ 17,  5,  8],  11,   1  {13,11,7,5,3}
        93  push %0         [ 17,  5,  8],  12,   1  {17,13,11,7,5,3}
        94  add %0, 2, %0   [ 19,  5,  8],  13,   1  {17,13,11,7,5,3}
        95  jmp 1           [ 19,  5,  8],   1,   1  {17,13,11,7,5,3}
        96  sub %0, 45, %2  [ 19,  5,-26],   2,   1  {17,13,11,7,5,3}
        97  jgt %2, 14      [ 19,  5,-26],   3,   1  {17,13,11,7,5,3}
        98  mov 3, %1       [ 19,  3,-26],   4,   1  {17,13,11,7,5,3}
        99  mul %1, %1, %2  [ 19,  3,  9],   5,   1  {17,13,11,7,5,3}
        100 sub %2, %0, %2  [ 19,  3,-10],   6,   1  {17,13,11,7,5,3}
        101 jgt %2, 11      [ 19,  3,-10],   7,   1  {17,13,11,7,5,3}
        102 mod %0, %1, %2  [ 19,  3,  1],   8,   1  {17,13,11,7,5,3}
        103 jeq %2, 12      [ 19,  3,  1],   9,   1  {17,13,11,7,5,3}
        104 add %1, 2, %1   [ 19,  5,  1],  10,   1  {17,13,11,7,5,3}
        105 jmp 4           [ 19,  5,  1],   4,   1  {17,13,11,7,5,3}
        106 mul %1, %1, %2  [ 19,  5, 25],   5,   1  {17,13,11,7,5,3}
        107 sub %2, %0, %2  [ 19,  5,  6],   6,   1  {17,13,11,7,5,3}
        108 jgt %2, 11      [ 19,  5,  6],  11,   1  {17,13,11,7,5,3}
        109 push %0         [ 19,  5,  6],  12,   1  {19,17,13,11,7,5,3}
        110 add %0, 2, %0   [ 21,  5,  6],  13,   1  {19,17,13,11,7,5,3}
        111 jmp 1           [ 21,  5,  6],   1,   1  {19,17,13,11,7,5,3}
        112 sub %0, 45, %2  [ 21,  5,-24],   2,   1  {19,17,13,11,7,5,3}
        113 jgt %2, 14      [ 21,  5,-24],   3,   1  {19,17,13,11,7,5,3}
        114 mov 3, %1       [ 21,  3,-24],   4,   1  {19,17,13,11,7,5,3}
        115 mul %1, %1, %2  [ 21,  3,  9],   5,   1  {19,17,13,11,7,5,3}
        116 sub %2, %0, %2  [ 21,  3,-12],   6,   1  {19,17,13,11,7,5,3}
        117 jgt %2, 11      [ 21,  3,-12],   7,   1  {19,17,13,11,7,5,3}
        118 mod %0, %1, %2  [ 21,  3,  0],   8,   1  {19,17,13,11,7,5,3}
        119 jeq %2, 12      [ 21,  3,  0],  12,   1  {19,17,13,11,7,5,3}
        120 add %0, 2, %0   [ 23,  3,  0],  13,   1  {19,17,13,11,7,5,3}
        121 jmp 1           [ 23,  3,  0],   1,   1  {19,17,13,11,7,5,3}
        122 sub %0, 45, %2  [ 23,  3,-22],   2,   1  {19,17,13,11,7,5,3}
        123 jgt %2, 14      [ 23,  3,-22],   3,   1  {19,17,13,11,7,5,3}
        124 mov 3, %1       [ 23,  3,-22],   4,   1  {19,17,13,11,7,5,3}
        125 mul %1, %1, %2  [ 23,  3,  9],   5,   1  {19,17,13,11,7,5,3}
        126 sub %2, %0, %2  [ 23,  3,-14],   6,   1  {19,17,13,11,7,5,3}
        127 jgt %2, 11      [ 23,  3,-14],   7,   1  {19,17,13,11,7,5,3}
        128 mod %0, %1, %2  [ 23,  3,  2],   8,   1  {19,17,13,11,7,5,3}
        129 jeq %2, 12      [ 23,  3,  2],   9,   1  {19,17,13,11,7,5,3}
        130 add %1, 2, %1   [ 23,  5,  2],  10,   1  {19,17,13,11,7,5,3}
        131 jmp 4           [ 23,  5,  2],   4,   1  {19,17,13,11,7,5,3}
        132 mul %1, %1, %2  [ 23,  5, 25],   5,   1  {19,17,13,11,7,5,3}
        133 sub %2, %0, %2  [ 23,  5,  2],   6,   1  {19,17,13,11,7,5,3}
        134 jgt %2, 11      [ 23,  5,  2],  11,   1  {19,17,13,11,7,5,3}
        135 push %0         [ 23,  5,  2],  12,   1  {23,19,17,13,11,7,5,3}
        136 add %0, 2, %0   [ 25,  5,  2],  13,   1  {23,19,17,13,11,7,5,3}
        137 jmp 1           [ 25,  5,  2],   1,   1  {23,19,17,13,11,7,5,3}
        138 sub %0, 45, %2  [ 25,  5,-20],   2,   1  {23,19,17,13,11,7,5,3}
        139 jgt %2, 14      [ 25,  5,-20],   3,   1  {23,19,17,13,11,7,5,3}
        140 mov 3, %1       [ 25,  3,-20],   4,   1  {23,19,17,13,11,7,5,3}
        141 mul %1, %1, %2  [ 25,  3,  9],   5,   1  {23,19,17,13,11,7,5,3}
        142 sub %2, %0, %2  [ 25,  3,-16],   6,   1  {23,19,17,13,11,7,5,3}
        143 jgt %2, 11      [ 25,  3,-16],   7,   1  {23,19,17,13,11,7,5,3}
        144 mod %0, %1, %2  [ 25,  3,  1],   8,   1  {23,19,17,13,11,7,5,3}
        145 jeq %2, 12      [ 25,  3,  1],   9,   1  {23,19,17,13,11,7,5,3}
        146 add %1, 2, %1   [ 25,  5,  1],  10,   1  {23,19,17,13,11,7,5,3}
        147 jmp 4           [ 25,  5,  1],   4,   1  {23,19,17,13,11,7,5,3}
        148 mul %1, %1, %2  [ 25,  5, 25],   5,   1  {23,19,17,13,11,7,5,3}
        149 sub %2, %0, %2  [ 25,  5,  0],   6,   1  {23,19,17,13,11,7,5,3}
        150 jgt %2, 11      [ 25,  5,  0],   7,   1  {23,19,17,13,11,7,5,3}
        151 mod %0, %1, %2  [ 25,  5,  0],   8,   1  {23,19,17,13,11,7,5,3}
        152 jeq %2, 12      [ 25,  5,  0],  12,   1  {23,19,17,13,11,7,5,3}
        153 add %0, 2, %0   [ 27,  5,  0],  13,   1  {23,19,17,13,11,7,5,3}
        154 jmp 1           [ 27,  5,  0],   1,   1  {23,19,17,13,11,7,5,3}
        155 sub %0, 45, %2  [ 27,  5,-18],   2,   1  {23,19,17,13,11,7,5,3}
        156 jgt %2, 14      [ 27,  5,-18],   3,   1  {23,19,17,13,11,7,5,3}
        157 mov 3, %1       [ 27,  3,-18],   4,   1  {23,19,17,13,11,7,5,3}
        158 mul %1, %1, %2  [ 27,  3,  9],   5,   1  {23,19,17,13,11,7,5,3}
        159 sub %2, %0, %2  [ 27,  3,-18],   6,   1  {23,19,17,13,11,7,5,3}
        160 jgt %2, 11      [ 27,  3,-18],   7,   1  {23,19,17,13,11,7,5,3}
        161 mod %0, %1, %2  [ 27,  3,  0],   8,   1  {23,19,17,13,11,7,5,3}
        162 jeq %2, 12      [ 27,  3,  0],  12,   1  {23,19,17,13,11,7,5,3}
        163 add %0, 2, %0   [ 29,  3,  0],  13,   1  {23,19,17,13,11,7,5,3}
        164 jmp 1           [ 29,  3,  0],   1,   1  {23,19,17,13,11,7,5,3}
        165 sub %0, 45, %2  [ 29,  3,-16],   2,   1  {23,19,17,13,11,7,5,3}
        166 jgt %2, 14      [ 29,  3,-16],   3,   1  {23,19,17,13,11,7,5,3}
        167 mov 3, %1       [ 29,  3,-16],   4,   1  {23,19,17,13,11,7,5,3}
        168 mul %1, %1, %2  [ 29,  3,  9],   5,   1  {23,19,17,13,11,7,5,3}
        169 sub %2, %0, %2  [ 29,  3,-20],   6,   1  {23,19,17,13,11,7,5,3}
        170 jgt %2, 11      [ 29,  3,-20],   7,   1  {23,19,17,13,11,7,5,3}
        171 mod %0, %1, %2  [ 29,  3,  2],   8,   1  {23,19,17,13,11,7,5,3}
        172 jeq %2, 12      [ 29,  3,  2],   9,   1  {23,19,17,13,11,7,5,3}
        173 add %1, 2, %1   [ 29,  5,  2],  10,   1  {23,19,17,13,11,7,5,3}
        174 jmp 4           [ 29,  5,  2],   4,   1  {23,19,17,13,11,7,5,3}
        175 mul %1, %1, %2  [ 29,  5, 25],   5,   1  {23,19,17,13,11,7,5,3}
        176 sub %2, %0, %2  [ 29,  5, -4],   6,   1  {23,19,17,13,11,7,5,3}
        177 jgt %2, 11      [ 29,  5, -4],   7,   1  {23,19,17,13,11,7,5,3}
        178 mod %0, %1, %2  [ 29,  5,  4],   8,   1  {23,19,17,13,11,7,5,3}
        179 jeq %2, 12      [ 29,  5,  4],   9,   1  {23,19,17,13,11,7,5,3}
        180 add %1, 2, %1   [ 29,  7,  4],  10,   1  {23,19,17,13,11,7,5,3}
        181 jmp 4           [ 29,  7,  4],   4,   1  {23,19,17,13,11,7,5,3}
        182 mul %1, %1, %2  [ 29,  7, 49],   5,   1  {23,19,17,13,11,7,5,3}
        183 sub %2, %0, %2  [ 29,  7, 20],   6,   1  {23,19,17,13,11,7,5,3}
        184 jgt %2, 11      [ 29,  7, 20],  11,   1  {23,19,17,13,11,7,5,3}
        185 push %0         [ 29,  7, 20],  12,   1  {29,23,19,17,13,11,7,5,3}
        186 add %0, 2, %0   [ 31,  7, 20],  13,   1  {29,23,19,17,13,11,7,5,3}
        187 jmp 1           [ 31,  7, 20],   1,   1  {29,23,19,17,13,11,7,5,3}
        188 sub %0, 45, %2  [ 31,  7,-14],   2,   1  {29,23,19,17,13,11,7,5,3}
        189 jgt %2, 14      [ 31,  7,-14],   3,   1  {29,23,19,17,13,11,7,5,3}
        190 mov 3, %1       [ 31,  3,-14],   4,   1  {29,23,19,17,13,11,7,5,3}
        191 mul %1, %1, %2  [ 31,  3,  9],   5,   1  {29,23,19,17,13,11,7,5,3}
        192 sub %2, %0, %2  [ 31,  3,-22],   6,   1  {29,23,19,17,13,11,7,5,3}
        193 jgt %2, 11      [ 31,  3,-22],   7,   1  {29,23,19,17,13,11,7,5,3}
        194 mod %0, %1, %2  [ 31,  3,  1],   8,   1  {29,23,19,17,13,11,7,5,3}
        195 jeq %2, 12      [ 31,  3,  1],   9,   1  {29,23,19,17,13,11,7,5,3}
        196 add %1, 2, %1   [ 31,  5,  1],  10,   1  {29,23,19,17,13,11,7,5,3}
        197 jmp 4           [ 31,  5,  1],   4,   1  {29,23,19,17,13,11,7,5,3}
        198 mul %1, %1, %2  [ 31,  5, 25],   5,   1  {29,23,19,17,13,11,7,5,3}
        199 sub %2, %0, %2  [ 31,  5, -6],   6,   1  {29,23,19,17,13,11,7,5,3}
        200 jgt %2, 11      [ 31,  5, -6],   7,   1  {29,23,19,17,13,11,7,5,3}
        201 mod %0, %1, %2  [ 31,  5,  1],   8,   1  {29,23,19,17,13,11,7,5,3}
        202 jeq %2, 12      [ 31,  5,  1],   9,   1  {29,23,19,17,13,11,7,5,3}
        203 add %1, 2, %1   [ 31,  7,  1],  10,   1  {29,23,19,17,13,11,7,5,3}
        204 jmp 4           [ 31,  7,  1],   4,   1  {29,23,19,17,13,11,7,5,3}
        205 mul %1, %1, %2  [ 31,  7, 49],   5,   1  {29,23,19,17,13,11,7,5,3}
        206 sub %2, %0, %2  [ 31,  7, 18],   6,   1  {29,23,19,17,13,11,7,5,3}
        207 jgt %2, 11      [ 31,  7, 18],  11,   1  {29,23,19,17,13,11,7,5,3}
        208 push %0         [ 31,  7, 18],  12,   1  {31,29,23,19,17,13,11,7,5,3}
        209 add %0, 2, %0   [ 33,  7, 18],  13,   1  {31,29,23,19,17,13,11,7,5,3}
        210 jmp 1           [ 33,  7, 18],   1,   1  {31,29,23,19,17,13,11,7,5,3}
        211 sub %0, 45, %2  [ 33,  7,-12],   2,   1  {31,29,23,19,17,13,11,7,5,3}
        212 jgt %2, 14      [ 33,  7,-12],   3,   1  {31,29,23,19,17,13,11,7,5,3}
        213 mov 3, %1       [ 33,  3,-12],   4,   1  {31,29,23,19,17,13,11,7,5,3}
        214 mul %1, %1, %2  [ 33,  3,  9],   5,   1  {31,29,23,19,17,13,11,7,5,3}
        215 sub %2, %0, %2  [ 33,  3,-24],   6,   1  {31,29,23,19,17,13,11,7,5,3}
        216 jgt %2, 11      [ 33,  3,-24],   7,   1  {31,29,23,19,17,13,11,7,5,3}
        217 mod %0, %1, %2  [ 33,  3,  0],   8,   1  {31,29,23,19,17,13,11,7,5,3}
        218 jeq %2, 12      [ 33,  3,  0],  12,   1  {31,29,23,19,17,13,11,7,5,3}
        219 add %0, 2, %0   [ 35,  3,  0],  13,   1  {31,29,23,19,17,13,11,7,5,3}
        220 jmp 1           [ 35,  3,  0],   1,   1  {31,29,23,19,17,13,11,7,5,3}
        221 sub %0, 45, %2  [ 35,  3,-10],   2,   1  {31,29,23,19,17,13,11,7,5,3}
        222 jgt %2, 14      [ 35,  3,-10],   3,   1  {31,29,23,19,17,13,11,7,5,3}
        223 mov 3, %1       [ 35,  3,-10],   4,   1  {31,29,23,19,17,13,11,7,5,3}
        224 mul %1, %1, %2  [ 35,  3,  9],   5,   1  {31,29,23,19,17,13,11,7,5,3}
        225 sub %2, %0, %2  [ 35,  3,-26],   6,   1  {31,29,23,19,17,13,11,7,5,3}
        226 jgt %2, 11      [ 35,  3,-26],   7,   1  {31,29,23,19,17,13,11,7,5,3}
        227 mod %0, %1, %2  [ 35,  3,  2],   8,   1  {31,29,23,19,17,13,11,7,5,3}
        228 jeq %2, 12      [ 35,  3,  2],   9,   1  {31,29,23,19,17,13,11,7,5,3}
        229 add %1, 2, %1   [ 35,  5,  2],  10,   1  {31,29,23,19,17,13,11,7,5,3}
        230 jmp 4           [ 35,  5,  2],   4,   1  {31,29,23,19,17,13,11,7,5,3}
        231 mul %1, %1, %2  [ 35,  5, 25],   5,   1  {31,29,23,19,17,13,11,7,5,3}
        232 sub %2, %0, %2  [ 35,  5,-10],   6,   1  {31,29,23,19,17,13,11,7,5,3}
        233 jgt %2, 11      [ 35,  5,-10],   7,   1  {31,29,23,19,17,13,11,7,5,3}
        234 mod %0, %1, %2  [ 35,  5,  0],   8,   1  {31,29,23,19,17,13,11,7,5,3}
        235 jeq %2, 12      [ 35,  5,  0],  12,   1  {31,29,23,19,17,13,11,7,5,3}
        236 add %0, 2, %0   [ 37,  5,  0],  13,   1  {31,29,23,19,17,13,11,7,5,3}
        237 jmp 1           [ 37,  5,  0],   1,   1  {31,29,23,19,17,13,11,7,5,3}
        238 sub %0, 45, %2  [ 37,  5, -8],   2,   1  {31,29,23,19,17,13,11,7,5,3}
        239 jgt %2, 14      [ 37,  5, -8],   3,   1  {31,29,23,19,17,13,11,7,5,3}
        240 mov 3, %1       [ 37,  3, -8],   4,   1  {31,29,23,19,17,13,11,7,5,3}
        241 mul %1, %1, %2  [ 37,  3,  9],   5,   1  {31,29,23,19,17,13,11,7,5,3}
        242 sub %2, %0, %2  [ 37,  3,-28],   6,   1  {31,29,23,19,17,13,11,7,5,3}
        243 jgt %2, 11      [ 37,  3,-28],   7,   1  {31,29,23,19,17,13,11,7,5,3}
        244 mod %0, %1, %2  [ 37,  3,  1],   8,   1  {31,29,23,19,17,13,11,7,5,3}
        245 jeq %2, 12      [ 37,  3,  1],   9,   1  {31,29,23,19,17,13,11,7,5,3}
        246 add %1, 2, %1   [ 37,  5,  1],  10,   1  {31,29,23,19,17,13,11,7,5,3}
        247 jmp 4           [ 37,  5,  1],   4,   1  {31,29,23,19,17,13,11,7,5,3}
        248 mul %1, %1, %2  [ 37,  5, 25],   5,   1  {31,29,23,19,17,13,11,7,5,3}
        249 sub %2, %0, %2  [ 37,  5,-12],   6,   1  {31,29,23,19,17,13,11,7,5,3}
        250 jgt %2, 11      [ 37,  5,-12],   7,   1  {31,29,23,19,17,13,11,7,5,3}
        251 mod %0, %1, %2  [ 37,  5,  2],   8,   1  {31,29,23,19,17,13,11,7,5,3}
        252 jeq %2, 12      [ 37,  5,  2],   9,   1  {31,29,23,19,17,13,11,7,5,3}
        253 add %1, 2, %1   [ 37,  7,  2],  10,   1  {31,29,23,19,17,13,11,7,5,3}
        254 jmp 4           [ 37,  7,  2],   4,   1  {31,29,23,19,17,13,11,7,5,3}
        255 mul %1, %1, %2  [ 37,  7, 49],   5,   1  {31,29,23,19,17,13,11,7,5,3}
        256 sub %2, %0, %2  [ 37,  7, 12],   6,   1  {31,29,23,19,17,13,11,7,5,3}
        257 jgt %2, 11      [ 37,  7, 12],  11,   1  {31,29,23,19,17,13,11,7,5,3}
        258 push %0         [ 37,  7, 12],  12,   1  {37,31,29,23,19,17,13,11,7,5,3}
        259 add %0, 2, %0   [ 39,  7, 12],  13,   1  {37,31,29,23,19,17,13,11,7,5,3}
        260 jmp 1           [ 39,  7, 12],   1,   1  {37,31,29,23,19,17,13,11,7,5,3}
        261 sub %0, 45, %2  [ 39,  7, -6],   2,   1  {37,31,29,23,19,17,13,11,7,5,3}
        262 jgt %2, 14      [ 39,  7, -6],   3,   1  {37,31,29,23,19,17,13,11,7,5,3}
        263 mov 3, %1       [ 39,  3, -6],   4,   1  {37,31,29,23,19,17,13,11,7,5,3}
        264 mul %1, %1, %2  [ 39,  3,  9],   5,   1  {37,31,29,23,19,17,13,11,7,5,3}
        265 sub %2, %0, %2  [ 39,  3,-30],   6,   1  {37,31,29,23,19,17,13,11,7,5,3}
        266 jgt %2, 11      [ 39,  3,-30],   7,   1  {37,31,29,23,19,17,13,11,7,5,3}
        267 mod %0, %1, %2  [ 39,  3,  0],   8,   1  {37,31,29,23,19,17,13,11,7,5,3}
        268 jeq %2, 12      [ 39,  3,  0],  12,   1  {37,31,29,23,19,17,13,11,7,5,3}
        269 add %0, 2, %0   [ 41,  3,  0],  13,   1  {37,31,29,23,19,17,13,11,7,5,3}
        270 jmp 1           [ 41,  3,  0],   1,   1  {37,31,29,23,19,17,13,11,7,5,3}
        271 sub %0, 45, %2  [ 41,  3, -4],   2,   1  {37,31,29,23,19,17,13,11,7,5,3}
        272 jgt %2, 14      [ 41,  3, -4],   3,   1  {37,31,29,23,19,17,13,11,7,5,3}
        273 mov 3, %1       [ 41,  3, -4],   4,   1  {37,31,29,23,19,17,13,11,7,5,3}
        274 mul %1, %1, %2  [ 41,  3,  9],   5,   1  {37,31,29,23,19,17,13,11,7,5,3}
        275 sub %2, %0, %2  [ 41,  3,-32],   6,   1  {37,31,29,23,19,17,13,11,7,5,3}
        276 jgt %2, 11      [ 41,  3,-32],   7,   1  {37,31,29,23,19,17,13,11,7,5,3}
        277 mod %0, %1, %2  [ 41,  3,  2],   8,   1  {37,31,29,23,19,17,13,11,7,5,3}
        278 jeq %2, 12      [ 41,  3,  2],   9,   1  {37,31,29,23,19,17,13,11,7,5,3}
        279 add %1, 2, %1   [ 41,  5,  2],  10,   1  {37,31,29,23,19,17,13,11,7,5,3}
        280 jmp 4           [ 41,  5,  2],   4,   1  {37,31,29,23,19,17,13,11,7,5,3}
        281 mul %1, %1, %2  [ 41,  5, 25],   5,   1  {37,31,29,23,19,17,13,11,7,5,3}
        282 sub %2, %0, %2  [ 41,  5,-16],   6,   1  {37,31,29,23,19,17,13,11,7,5,3}
        283 jgt %2, 11      [ 41,  5,-16],   7,   1  {37,31,29,23,19,17,13,11,7,5,3}
        284 mod %0, %1, %2  [ 41,  5,  1],   8,   1  {37,31,29,23,19,17,13,11,7,5,3}
        285 jeq %2, 12      [ 41,  5,  1],   9,   1  {37,31,29,23,19,17,13,11,7,5,3}
        286 add %1, 2, %1   [ 41,  7,  1],  10,   1  {37,31,29,23,19,17,13,11,7,5,3}
        287 jmp 4           [ 41,  7,  1],   4,   1  {37,31,29,23,19,17,13,11,7,5,3}
        288 mul %1, %1, %2  [ 41,  7, 49],   5,   1  {37,31,29,23,19,17,13,11,7,5,3}
        289 sub %2, %0, %2  [ 41,  7,  8],   6,   1  {37,31,29,23,19,17,13,11,7,5,3}
        290 jgt %2, 11      [ 41,  7,  8],  11,   1  {37,31,29,23,19,17,13,11,7,5,3}
        291 push %0         [ 41,  7,  8],  12,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        292 add %0, 2, %0   [ 43,  7,  8],  13,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        293 jmp 1           [ 43,  7,  8],   1,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        294 sub %0, 45, %2  [ 43,  7, -2],   2,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        295 jgt %2, 14      [ 43,  7, -2],   3,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        296 mov 3, %1       [ 43,  3, -2],   4,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        297 mul %1, %1, %2  [ 43,  3,  9],   5,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        298 sub %2, %0, %2  [ 43,  3,-34],   6,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        299 jgt %2, 11      [ 43,  3,-34],   7,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        300 mod %0, %1, %2  [ 43,  3,  1],   8,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        301 jeq %2, 12      [ 43,  3,  1],   9,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        302 add %1, 2, %1   [ 43,  5,  1],  10,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        303 jmp 4           [ 43,  5,  1],   4,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        304 mul %1, %1, %2  [ 43,  5, 25],   5,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        305 sub %2, %0, %2  [ 43,  5,-18],   6,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        306 jgt %2, 11      [ 43,  5,-18],   7,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        307 mod %0, %1, %2  [ 43,  5,  3],   8,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        308 jeq %2, 12      [ 43,  5,  3],   9,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        309 add %1, 2, %1   [ 43,  7,  3],  10,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        310 jmp 4           [ 43,  7,  3],   4,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        311 mul %1, %1, %2  [ 43,  7, 49],   5,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        312 sub %2, %0, %2  [ 43,  7,  6],   6,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        313 jgt %2, 11      [ 43,  7,  6],  11,   1  {41,37,31,29,23,19,17,13,11,7,5,3}
        314 push %0         [ 43,  7,  6],  12,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        315 add %0, 2, %0   [ 45,  7,  6],  13,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        316 jmp 1           [ 45,  7,  6],   1,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        317 sub %0, 45, %2  [ 45,  7,  0],   2,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        318 jgt %2, 14      [ 45,  7,  0],   3,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        319 mov 3, %1       [ 45,  3,  0],   4,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        320 mul %1, %1, %2  [ 45,  3,  9],   5,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        321 sub %2, %0, %2  [ 45,  3,-36],   6,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        322 jgt %2, 11      [ 45,  3,-36],   7,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        323 mod %0, %1, %2  [ 45,  3,  0],   8,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        324 jeq %2, 12      [ 45,  3,  0],  12,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        325 add %0, 2, %0   [ 47,  3,  0],  13,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        326 jmp 1           [ 47,  3,  0],   1,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        327 sub %0, 45, %2  [ 47,  3,  2],   2,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        328 jgt %2, 14      [ 47,  3,  2],  14,   1  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        329 end             [ 47,  3,  2],  14,   0  {43,41,37,31,29,23,19,17,13,11,7,5,3}
        ------- output stack
        3,5,7,11,13,17,19,23,29,31,37,41,43,
        ------- real result
        3,5,7,11,13,17,19,23,29,31,37,41,43,
