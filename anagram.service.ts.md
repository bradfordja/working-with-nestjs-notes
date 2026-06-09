Yes. In NestJS, this logic belongs in a Service because it contains business logic. The Controller would simply call the service.

NestJS Service Version

anagram.service.ts

import { Injectable } from '@nestjs/common';
@Injectable()
export class AnagramService {
  groupAnagrams(strs: string[]): string[][] {
    if (!strs || strs.length === 0) {
      return [];
    }
    const map = new Map<string, string[]>();
    for (const word of strs) {
      // Sort characters to create grouping key
      const key = word
        .split('')
        .sort()
        .join('');
      if (!map.has(key)) {
        map.set(key, []);
      }
      map.get(key)?.push(word);
    }
    return Array.from(map.values());
  }
}

⸻

Controller Example

anagram.controller.ts

import { Body, Controller, Post } from '@nestjs/common';
import { AnagramService } from './anagram.service';
@Controller('anagrams')
export class AnagramController {
  constructor(
    private readonly anagramService: AnagramService,
  ) {}
  @Post('/group')
  group(@Body() body: { words: string[] }) {
    return this.anagramService.groupAnagrams(
      body.words,
    );
  }
}

⸻

Module Registration

anagram.module.ts

import { Module } from '@nestjs/common';
import { AnagramController } from './anagram.controller';
import { AnagramService } from './anagram.service';
@Module({
  controllers: [AnagramController],
  providers: [AnagramService],
})
export class AnagramModule {}

⸻

App Module

app.module.ts

import { Module } from '@nestjs/common';
import { AnagramModule } from './anagram/anagram.module';
@Module({
  imports: [AnagramModule],
})
export class AppModule {}

⸻

API Request

POST

POST /anagrams/group
Content-Type: application/json

Request:

{
  "words": [
    "eat",
    "tea",
    "tan",
    "ate",
    "nat",
    "bat"
  ]
}

Response:

[
  ["eat","tea","ate"],
  ["tan","nat"],
  ["bat"]
]

⸻

Senior NestJS Version (Using DTO)

In a real enterprise application, you’d use DTOs and validation.

group-anagram.dto.ts

import {
  IsArray,
  IsString,
} from 'class-validator';
export class GroupAnagramDto {
  @IsArray()
  @IsString({ each: true })
  words: string[];
}

Controller

@Post('/group')
group(
  @Body()
  dto: GroupAnagramDto,
) {
  return this.anagramService.groupAnagrams(
    dto.words,
  );
}

⸻

Senior Interview Question

Q: What is the Time Complexity of this solution?

Answer:

For each word:

Convert word to char array = O(k)
Sort characters         = O(k log k)

Where:

n = number of strings
k = average word length

Total:

O(n * k log k)

Space Complexity:

O(n * k)

because all strings are stored in the HashMap groups.

⸻

Even More Senior Optimization

Instead of sorting each word, create a frequency signature.

Example:

eat -> a1e1t1
tea -> a1e1t1
ate -> a1e1t1

Complexity

O(n * k)

instead of

O(n * k log k)

Example key generation:

const counts = new Array(26).fill(0);
for (const ch of word) {
  counts[ch.charCodeAt(0) - 97]++;
}
const key = counts.join('#');

This is the version many senior-level interviewers expect when discussing performance optimization of the Anagram Grouping problem.