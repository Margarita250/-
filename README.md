// ============================================================
// 1. TestingPlatform.Application/Dtos/AttemptDto.cs
// ============================================================
namespace TestingPlatform.Application.Dtos;

/// <summary>
/// Модель попытки прохождения теста
/// </summary>
public class AttemptDto
{
    /// <summary>
    /// Идентификатор попытки
    /// </summary>
    public int Id { get; set; }

    /// <summary>
    /// Время начала попытки
    /// </summary>
    public DateTimeOffset StartedAt { get; set; }

    /// <summary>
    /// Время окончания попытки
    /// </summary>
    public DateTimeOffset? SubmittedAt { get; set; }

    /// <summary>
    /// Количество баллов за попытку
    /// </summary>
    public int? Score { get; set; }

    /// <summary>
    /// По какому тесту была попытка
    /// </summary>
    public int TestId { get; set; }

    /// <summary>
    /// Какой студент выполнял попытку
    /// </summary>
    public int StudentId { get; set; }
}


// ============================================================
// 2. TestingPlatform.Infrastructure/Mappings/AttemptProfile.cs
// ============================================================
using AutoMapper;
using TestingPlatform.Application.Dtos;
using TestingPlatform.Domain.Models;

namespace TestingPlatform.Infrastructure.Mappings;

public class AttemptProfile : Profile
{
    public AttemptProfile()
    {
        CreateMap<AttemptDto, Attempt>();
    }
}


// ============================================================
// 3. TestingPlatform.Application/Interfaces/IAttemptRepository.cs
// ============================================================
using TestingPlatform.Application.Dtos;

namespace TestingPlatform.Application.Interfaces;

public interface IAttemptRepository
{
    /// <summary>
    /// Создать новую попытку
    /// </summary>
    /// <param name="attemptDto">Модель создания новой попытки</param>
    /// <returns>Идентификатор новой попытки</returns>
    Task<int> CreateAsync(AttemptDto attemptDto);

    /// <summary>
    /// Обновить попытку (завершить)
    /// </summary>
    /// <param name="attemptDto">Модель попытки</param>
    Task UpdateAsync(AttemptDto attemptDto);
}


// ============================================================
// 4. TestingPlatform.Infrastructure/Repositories/AttemptRepository.cs
// ============================================================
using AutoMapper;
using Microsoft.EntityFrameworkCore;
using TestingPlatform.Application.Dtos;
using TestingPlatform.Application.Interfaces;
using TestingPlatform.Domain.Models;
using TestingPlatform.Infrastructure.Exceptions;

namespace TestingPlatform.Infrastructure.Repositories;

public class AttemptRepository(
    AppDbContext appDbContext,
    IMapper mapper,
    ITestRepository testRepository) : IAttemptRepository
{
    public async Task<int> CreateAsync(AttemptDto attemptDto)
    {
        // 1. Проверяем, что тест существует
        var test = await testRepository.GetByIdAsync(attemptDto.TestId);
        if (test is null)
            throw new EntityNotFoundException("Тест не найден.");

        // 2. Проверяем, что студент существует
        var student = await appDbContext.Students
            .AsNoTracking()
            .FirstOrDefaultAsync(s => s.Id == attemptDto.StudentId);
        if (student == null)
            throw new EntityNotFoundException("Студент не найден.");

        // 3. Проверяем, что тест опубликован
        if (!test.IsPublic)
            throw new InvalidOperationException("Тест не доступен");

        // 4. Проверяем, что тест доступен студенту
        var availableTests = await testRepository.GetAllForStudent(attemptDto.StudentId);
        if (availableTests.All(t => t.Id != attemptDto.TestId))
            throw new InvalidOperationException("Доступ запрещен");

        var attempt = mapper.Map<Attempt>(attemptDto);

        // 5. Если тест повторяемый и попытки неограничены — сразу создаём попытку
        if (test is { IsRepeatable: true, MaxAttempts: null })
            return await CreateAttemptAsync(attempt);

        // 6. Получаем предыдущие попытки студента по этому тесту
        var lastAttempts = await appDbContext.Attempts
            .Where(a => a.StudentId == attemptDto.StudentId && a.TestId == attemptDto.TestId)
            .ToListAsync();

        // 7. Обрабатываем незавершённые попытки
        var inProgress = lastAttempts.FirstOrDefault(a => a.SubmittedAt == null);
        if (inProgress != null)
        {
            if (test.DurationMinutes.HasValue)
            {
                var expiresAt = inProgress.StartedAt.AddMinutes(test.DurationMinutes.Value);
                if (DateTimeOffset.UtcNow < expiresAt)
                {
                    throw new InvalidOperationException(
                        "Есть незавершённая попытка, время выполнения ещё не истекло.");
                }
                // Если время истекло — разрешаем начать новую попытку
            }
            else
            {
                throw new InvalidOperationException(
                    "Есть незавершённая попытка. Тест не имеет ограничений по времени, поэтому новую попытку начать нельзя.");
            }
        }

        // 8. Неповторяемый тест уже был пройден
        if (!test.IsRepeatable && lastAttempts.Count > 0)
            throw new InvalidOperationException("Тест нельзя пройти более одного раза");

        // 9. Исчерпано количество попыток
        if (test.IsRepeatable && lastAttempts.Count >= test.MaxAttempts)
            throw new InvalidOperationException("Исчерпано количество прохождений теста");

        return await CreateAttemptAsync(attempt);
    }

    /// <summary>
    /// Внутренний метод сохранения новой попытки
    /// </summary>
    private async Task<int> CreateAttemptAsync(Attempt attempt)
    {
        attempt.StartedAt = DateTimeOffset.Now;
        attempt.Score = 0;

        var entityEntry = await appDbContext.Attempts.AddAsync(attempt);
        await appDbContext.SaveChangesAsync();

        return entityEntry.Entity.Id;
    }

    public async Task UpdateAsync(AttemptDto attemptDto)
    {
        // 1. Находим попытку вместе с ответами
        var attempt = await appDbContext.Attempts
            .Include(a => a.UserAttemptAnswers)
            .FirstOrDefaultAsync(a => a.Id == attemptDto.Id);

        if (attempt is null)
            throw new EntityNotFoundException("Попытка не найдена");

        // 2. Нельзя завершить уже сданную попытку
        if (attempt.SubmittedAt != null)
            throw new InvalidOperationException("Нельзя завершить уже сданную попытку.");

        // 3. Завершаем попытку и считаем баллы
        attempt.SubmittedAt = DateTimeOffset.Now;
        attempt.Score = attempt.UserAttemptAnswers.Sum(ua => ua.ScoreAwarded);

        // 4. Получаем тест
        var test = await appDbContext.Tests
            .AsNoTracking()
            .FirstOrDefaultAsync(t => t.Id == attempt.TestId);

        // 5. Обновляем / создаём результат теста
        var testResult = await appDbContext.TestResults
            .Include(tr => tr.Attempt)
            .FirstOrDefaultAsync(tr => tr.TestId == attempt.TestId
                                       && tr.StudentId == attempt.StudentId);

        if (testResult == null)
        {
            var newTestResult = new TestResult
            {
                AttemptId = attempt.Id,
                StudentId = attempt.StudentId,
                TestId = attempt.TestId,
                Passed = test!.PassingScore == null || test.PassingScore <= attempt.Score
            };
            await appDbContext.TestResults.AddAsync(newTestResult);
        }
        else
        {
            if (testResult.Attempt.Score < attempt.Score)
            {
                testResult.AttemptId = attempt.Id;
                testResult.Passed = test!.PassingScore == null || test.PassingScore <= attempt.Score;
            }
        }

        await appDbContext.SaveChangesAsync();
    }
}


// ============================================================
// 5. Program.cs — регистрация репозитория
// ============================================================
builder.Services.AddScoped<IAttemptRepository, AttemptRepository>();


// ============================================================
// 6. practice/Requests/Attempt/CreateAttemptRequest.cs
// ============================================================
namespace practice.Requests.Attempt;

/// <summary>
/// Модель начала попытки
/// </summary>
public class CreateAttemptRequest
{
    /// <summary>
    /// Идентификатор теста
    /// </summary>
    public int TestId { get; set; }

    //TODO: удалить после добавления авторизации
    /// <summary>
    /// [Временно (после добавления авторизации будет удален)] Идентификатор студента
    /// </summary>
    public int StudentId { get; set; }
}


// ============================================================
// 7. practice/Requests/Attempt/UpdateAttemptRequest.cs
// ============================================================
namespace practice.Requests.Attempt;

public class UpdateAttemptRequest
{
    /// <summary>
    /// Идентификатор попытки
    /// </summary>
    public int Id { get; set; }

    /// <summary>
    /// Идентификатор студента
    /// </summary>
    public int StudentId { get; set; }
}


// ============================================================
// 8. practice/Mappings/AttemptProfile.cs
// ============================================================
using AutoMapper;
using practice.Requests.Attempt;
using TestingPlatform.Application.Dtos;

namespace practice.Mappings;

public class AttemptProfile : Profile
{
    public AttemptProfile()
    {
        CreateMap<CreateAttemptRequest, AttemptDto>();
        CreateMap<UpdateAttemptRequest, AttemptDto>();
    }
}


// ============================================================
// 9. practice/Controllers/AttemptsController.cs
// ============================================================
using AutoMapper;
using Microsoft.AspNetCore.Mvc;
using practice.Requests.Attempt;
using TestingPlatform.Application.Dtos;
using TestingPlatform.Application.Interfaces;

namespace practice.Controllers;

[ApiController]
[Route("api/[controller]")]
[ProducesResponseType(StatusCodes.Status401Unauthorized)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status500InternalServerError)]
public class AttemptsController(
    IAttemptRepository attemptRepository,
    IMapper mapper) : ControllerBase
{
    /// <summary>
    /// Начать попытку
    /// </summary>
    [HttpPost]
    [ProducesResponseType(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> CreateAttempt([FromBody] CreateAttemptRequest attempt)
    {
        var attemptDto = mapper.Map<AttemptDto>(attempt);

        var attemptId = await attemptRepository.CreateAsync(attemptDto);

        return StatusCode(StatusCodes.Status201Created, new { Id = attemptId });
    }

    /// <summary>
    /// Закончить попытку
    /// </summary>
    [HttpPut]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> UpdateAttempt([FromBody] UpdateAttemptRequest attempt)
    {
        var attemptDto = mapper.Map<AttemptDto>(attempt);

        await attemptRepository.UpdateAsync(attemptDto);

        return StatusCode(StatusCodes.Status200OK);
    }
}
